---
layout: post
title: 在ASPNET-Web-API中使用Redis
category: dotnet
tags: Redis,ASPNET
date: 2014-05-01
---



本文将介绍如何在ASP.NET Web API应用程序中使用Redis作为数据存储。利用*ServiceStack.Redis*库以及它*强类型*的Redis 客户端，实现如何建模和存储一对多的关系，以及使用Autofac的Web API依赖注入的功能，实现将资源库(respositories)注入到控制器(controllers)中。 

#### 客户端(Client)类库

写作本文时有两个流行和活跃的C#版Redis客户端类库：

*   [ServiceStack.Redis][1]-由Demis Bellot利用ServiceStack之名创建，且基于 [Miguel  
    de Icaza][2]的 [redis-sharp][3] 项目。
*   [BookSleeve][4] – 由Mark Gravell维护，而且据我所知它正被[Stack Exchange][5]使用。

在做出选择之前，建议两个都试用下，再决定哪个API和功能更适合你的项目。 BookSleeve包含非阻塞（异步）API，提供线程安全的连接对象，而ServiceStack的实现提供JSON序列化功能、类似于客户端工厂的连接池以及使用约定简化POCO(Plain Old CLR Object)对象的持久化。

本文将使用*ServiceStack.Redis*,但是记住*BookSleeve*已经在大型的Web应用中使用且证明性能表现优良。

#### nutshell中的Redis

你读到本文时很有可能对Redis有所了解了。如果你是一个对ASP.NET Web API 集成感兴趣的经验丰富的Redis用户，那么你可以安全的跳到下一部分。

为了有效使用Redis且避免潜在的陷阱，你得了解她是如何工作的以及它同关系型数据库的区别。强烈建议阅读一本书或者关于它的在线材料。

简言之，Redis是一个支持持久化的键值对内存数据库。*基于内存（in-memory）*和 *键值对(key-value)*听起来很像一个内存缓存-实际上你可以将Redis看做是一个专业的而且更加先进的内存缓存。

相比于传统数据库，它的主要优势是它直接在高速运转的内存中存储和获取数据-这意味着它实际上速度很快。

Redis是简单且专业的-不同于关系型数据库，它不提供任何类似于表结构的抽象以及关系能力。但是，它提供五个基本数据类型，以及处理这些类型（存储值）的特定操作。这也是为什么它有时会被当做一个 *数据结构服务器（data structure server）* 的原因：

*   *字符串(strings)*-最基本且最原子的类型，用于储存任何数据（整数，序列化POCO对象等等）
*   *列表(lists)*-根据插入的顺序排序后的字符串列表。
*   *集合(sets)*-字符串的逻辑集合。
*   *哈希表(hashes)*-字符串类型的key和字符串类型的value之间的映射。
*   *有序集合(sorted sets)*-类似于集合，但是每个元素都对应一个用于排序的得分。

专用的命令集有：

*   字符串-SET, INCR, APPEND, INCRBY, STRLEN, SETBIT,
*   列表- LPUSH, LPOP, LTRIM, LINSERT,
*   集合-SADD, SDIFF, SINTER, SUNION, etc.

希望这可以让你对Redis有个基本的了解。

#### 为什么要使用Redis

Redis在应用程序能否起作用以及作用多大，取决于应用程序的体系结构，数据量，数据复杂度和有经验的负载。若运用得当，Redis将会带来重大的性能提升而且将为大规模应用程序提供强有力的支持。

下面是我认为的一些使用场景：

*   作为一个主数据库，
*   作为众多数据库中的一个，例如存储小数据量但是频繁访问的数据，
*   作为一个高性能的领域模型只读视图，
*   作为一个缓存。

值得注意的是Redis工作在内存中，所以第一个场景是很极端的，只有在你的数据量很小或者拥有足够多的RAM的情况下才可行。

由于本文主要关注ASP.NET Web API 集成，而不是架构方面，因此选择这种场景进行介绍。

#### Redis在ASP.NET Web API应用程序中的应用

本文将以空ASP.NET Web API应用程序为起点，使用2个第三方类库：

*   ServiceStack.Redis-C# Redis客户端,
*   Autofac-集成Web API的依赖注入容器.

显然也需要一个处于运行状态的Redis服务实例。如果没有Redis服务，那么你可以下载MS Tech提供的Windows 移植版本。请注意该移植版本还不是正式的产品（为此你需要使用其中一个官方包），但在开发场景中是有效的。

#### 模型

针对本例，考虑以下需求：

*   API应该提供存储客户、获取客户详情以及获取系统中的所有客户列表的功能。
*   客户端可以订购由多个商品组成的订单。
*   API应该提供一个最畅销的N个商品的列表

下面是我们设计的模型：

    public class Customer
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public IList<Guid> Orders { get; set; }
        public Address Address { get; set; }
    }

    

Properly defining your data model will help you use Redis in an efficient way. Redis stores values as byte blobs internally and *ServiceStack.Redis* will serialize the whole object graph for us. Thus it is important that we define aggregate boundaries.As you can see Address is a *value object* and will be persisted and retrieved as a part of Customer *aggregate*,while *Orders* property is a list of ids.

    
    
    public class Order
    {
        public Guid Id { get; set; }
        public Guid UserId { get; set; }
        public IList<OrderLine> Lines { get; set; }
    }

    public class OrderLine
    {
        public string Item { get; set; }
        public int Quantity { get; set; }
        public decimal TotalAmount { get; set; }
    }

    public class Address
    {
        public string Line1 { get; set; }
        public string Line2 { get; set; }
        public string City { get; set; }
    }


现在定义资源库接口：

    public interface ICustomerRepository
    {
        IList<Customer> GetAll();
        Customer Get(Guid id);
        Customer Store(Customer customer);
    }

    public interface IOrderRepository
    {
        IList<Order> GetCustomerOrders(Guid customerId);
        IList<Order> StoreAll(Customer customer, IList<Order> orders);
        Order Store(Customer customer, Order order);
        IDictionary<string, double> GetBestSellingItems(int count);
    }


实现如下：

    public class CustomerRepository : ICustomerRepository
    {
        private readonly IRedisClient _redisClient;

        public CustomerRepository(IRedisClient redisClient)
        {
            _redisClient = redisClient;
        }

        public IList<Customer> GetAll()
        {
            using (var typedClient = _redisClient.GetTypedClient<Customer>())
            {
                return typedClient.GetAll();
            }
        }

        public Customer Get(Guid id)
        {
            using (var typedClient = _redisClient.GetTypedClient<Customer>())
            {
                return typedClient.GetById(id);
            }
        }

        public Customer Store(Customer customer)
        {
            using (var typedClient = _redisClient.GetTypedClient<Customer>())
            {
                if (customer.Id == default(Guid))
                {
                    customer.Id = Guid.NewGuid();
                }
                return typedClient.Store(customer);
            }
        }
    }

    public class OrderRepository : IOrderRepository
    {
        private readonly IRedisClient _redisClient;

        public OrderRepository(IRedisClient redisClient)
        {
            _redisClient = redisClient;
        }

        public IList<Order> GetCustomerOrders(Guid customerId)
        {
            using (var orderClient = _redisClient.GetTypedClient<Order>())
            {
                var orderIds = _redisClient.GetAllItemsFromSet(RedisKeys
                            .GetCustomerOrdersReferenceKey(customerId));
                IList<Order> orders = orderClient.GetByIds(orderIds);
                return orders;
            }
        }

        public IList<Order> StoreAll(Customer customer, IList<Order> orders)
        {
            foreach (var order in orders)
            {
                if (order.Id == default(Guid))
                {
                    order.Id = Guid.NewGuid();
                }
                order.CustomerId = customer.Id;
                if (!customer.Orders.Contains(order.Id))
                {
                    customer.Orders.Add(order.Id);
                }

                order.Lines.ForEach(l=>_redisClient
                    .IncrementItemInSortedSet(RedisKeys.BestSellingItems,
                                                                     (string) l.Item, (long) l.Quantity));
            }
            var orderIds = orders.Select(o => o.Id.ToString()).ToList();
            using (var transaction = _redisClient.CreateTransaction())
            {
                transaction.QueueCommand(c => c.Store(customer));
                transaction.QueueCommand(c => c.StoreAll(orders));
                transaction.QueueCommand(c => c.AddRangeToSet(RedisKeys
                    .GetCustomerOrdersReferenceKey(customer.Id),
                    orderIds));
                transaction.Commit();
            }

            return orders;
        }

        public Order Store(Customer customer, Order order)
        {
            IList<Order> result = StoreAll(customer, new List<Order>() { order });
            return result.FirstOrDefault();
        }

        public IDictionary<string, double> GetBestSellingItems(int count)
        {
            return _redisClient
                .GetRangeWithScoresFromSortedSetDesc(RedisKeys.BestSellingItems,
                0, count - 1);
        }
    }


可以看到资源库（respositories）实现了专有的操作。可以利用Redis有序集类型有效存储和获取最畅销商品列表。

值得注意的是我们如何实现Customer-*Orders的关系。我们在一个专属集合中存储了客户的订单（它们的Id），因此不需要取出整个Customer实体就能够快速获取到它们。

#### 客户端和连接生命周期管理

我们面临的其中一个挑战是连接/客户端生命周期管理。正如你已经知道的那样，Web API提供一个可扩展的依赖注入机制，它可以用来对每个请求注入和处理依赖。如果不打算从零开始自己编写 *IDependencyResolver* 的实现（当然这也是一种选择），我们可以使用.NET DI库比如Ninject, StructureMap, Unity, Windsor 或者 Autofac。最后一个DI库是我个人喜欢的，而且很好的集成了Web API，这也是为什么我在本例中使用它的缘故。

ServiceStack.Redis 拥有 *IRedisClient*工厂，即 *客户端*管理器（client managers）:

*   BasicRedisClientManager-支持负载均衡的客户端工厂，
*   PooledRedisClientManager-支持负载均衡和连接池的客户端工厂-实际工作中很有用。
*   ShardedRedisClientManager-利用一致性哈希算法提供客户端连接分片（sharding）。

由于这些类库是线程安全（thread-safe）的，因此可以在所有的请求中使用一个工厂实例。

    public class ApiApplication : System.Web.HttpApplication {
        public IRedisClientsManager ClientsManager;
        private const string RedisUri = "localhost";

        protected void Application_Start()
        {
            ClientsManager = new PooledRedisClientManager(RedisUri);

            AreaRegistration.RegisterAllAreas();

            WebApiConfig.Register(GlobalConfiguration.Configuration);
            ConfigureDependencyResolver(GlobalConfiguration.Configuration);

            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            RouteConfig.RegisterRoutes(RouteTable.Routes);
        }

        private void ConfigureDependencyResolver(HttpConfiguration configuration)
        {
            var builder = new ContainerBuilder();
            builder.RegisterApiControllers(Assembly.GetExecutingAssembly())
                .PropertiesAutowired();

            builder.RegisterType<CustomerRepository>()
                .As<ICustomerRepository>()
                .PropertiesAutowired()
                .InstancePerApiRequest();

            builder.RegisterType<OrderRepository>()
                .As<IOrderRepository>()
                .PropertiesAutowired()
                .InstancePerApiRequest();

            builder.Register<IRedisClient>(c => ClientsManager.GetClient())
                .InstancePerApiRequest();

            configuration.DependencyResolver
                = new AutofacWebApiDependencyResolver(builder.Build());
        }

        protected void Application_OnEnd()
        {
            ClientsManager.Dispose();
        }
    }


我们使用池连接管理器作为*IRedisClientsManager*的实现。每当一个请求触发时，就会获得一个新的客户端实例，并且注入到资源库中而且在请求结束时进行处理。

#### 控制器

既然有了资源库，那么我们来实现控制器-一个用于添加和获取顾客另一个用于管理订单。

    public class CustomersController : ApiController
    {
        public ICustomerRepository CustomerRepository { get; set; }

        public IOrderRepository OrderRepository { get; set; }

        public IQueryable<Customer> GetAll()
        {
            return CustomerRepository.GetAll().AsQueryable();
        }

        public Customer Get(Guid id)
        {
            var customer = CustomerRepository.Get(id);
            if (customer == null)
            {
                throw new HttpResponseException(HttpStatusCode.NotFound);
            }

            return customer;
        }

        public HttpResponseMessage Post([FromBody] Customer customer)
        {
            var result = CustomerRepository.Store(customer);
            return Request.CreateResponse(HttpStatusCode.Created, result);
        }

        public HttpResponseMessage Put(Guid id, [FromBody] Customer customer)
        {
            var existingEntity = CustomerRepository.Get(id);
            if (existingEntity == null)
            {
                throw new HttpResponseException(HttpStatusCode.NotFound);
            }
            customer.Id = id;
            CustomerRepository.Store(customer);
            return Request.CreateResponse(HttpStatusCode.NoContent);
        }
    }

    public class OrdersController : ApiController
    {
        public IOrderRepository OrderRepository { get; set; }
        public ICustomerRepository CustomerRepository { get; set; }

        public HttpResponseMessage Post([FromBody] Order order)
        {
            var customer = CustomerRepository.Get(order.CustomerId);
            var result = OrderRepository.Store(customer, order);
            return Request.CreateResponse(HttpStatusCode.Created, result);
        }

        [ActionName("top")]
        [HttpGet]
        public IDictionary<string, double> GetBestSellingItems(int count)
        {
            return OrderRepository.GetBestSellingItems(count);
        }

        [ActionName("customer")]
        [HttpGet]
        public IList<Order> GetCustomerOrders(Guid id)
        {
            return OrderRepository.GetCustomerOrders(id);
        }
    }


这就是本文的所有内容，本文使用Redis作为数据存储而且依赖将自动连接。

源代码托管在[Bitbucket][6].

原文链接：<http://www.piotrwalat.net/using-redis-with-asp-net-web-api/?utm_source=tuicool>

 [1]: https://github.com/ServiceStack/ServiceStack.Redis
 [2]: http://twitter.com/migueldeicaza
 [3]: http://github.com/migueldeicaza/redis-sharp
 [4]: https://code.google.com/p/booksleeve/
 [5]: http://marcgravell.blogspot.ie/2011/04/async-redis-await-booksleeve.html
 [6]: https://bitbucket.org/pwalat/piotr.rediswebapi
