# Modernizing Enterprise Systems via Decomposition to Microservices on MongoDB

In my role as the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most formidable projects I've encountered involved the transformation of an aging monolithic ERP system into a modern microservices architecture. This outdated monolith had become a costly burden to maintain and was stifling my client's ability to innovate.

Thankfully, this was not an unfamiliar challenge, given our years of providing custom [Software Development Services in Boston](https://hybridwebagency.com/boston-ma/best-software-development-company/). I'm well-acquainted with the obstacles organizations face when dealing with rigid and resistant-to-change systems. This is why I'm excited to share the systematic approach we took to restructure the data model and break down the monolith, both at the code and database levels.

Throughout the migration process, my team and I uncovered some lesser-known best practices for shaping domain entities within a document database like MongoDB. These practices ensure seamless support for fully independent microservices. If you've ever wondered about future-proofing your database architecture for the cloud while preserving historical data, you're in for some enlightening strategies.

By the end of this article, you'll have a comprehensive blueprint for migrating your legacy systems to a modern architecture. I'll provide a wealth of practical insights to help you avoid common pitfalls and accelerate the delivery of new features to your customers. Let's dive into the details!

## The Advantages of Adopting a Microservices Architecture

The adoption of a microservices architecture offers several advantages over the traditional monolithic approach. Microservices allow for independent deployments, enabling rapid development and feature releases without disrupting the entire application.

Additionally, individual services can be developed using different programming languages and frameworks. This flexibility empowers organizations to choose the most suitable technologies for each specific domain. For instance, a recommendation engine might leverage Python's machine learning libraries, while the user interface is built using React.

This decoupling of concerns facilitates the work of specialized teams on separate services. It encourages a culture of experimentation, as ideas can be quickly validated through prototypes before committing to a full-scale monolithic overhaul. Moreover, new team members can make meaningful contributions to a single service aligned with their expertise.

In the ever-evolving technology landscape, trends come and go. Microservices act as a safeguard against these shifts, as replacements only involve small, isolated portions of the system, rather than a complete rewrite. Clearly defined interfaces make migrating components to new implementations a seamless process, ensuring a smooth transition.

#### Independent Scalability

Efficient resource management is achieved when services scale independently in response to demand. The frontend API gateway, for example, routes traffic based on URLs and can securely deploy behind a load balancer to handle peak loads. This means that during peak seasons, such as holidays, only the order processing service needs additional servers, without affecting the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service layer results in significant cost savings by right-sizing resources accurately. Idle support microservices, such as user profiles, no longer require costly overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model

### Understanding Entity Relationships

The initial step in transitioning to microservices is a thorough analysis of how entities interconnect within the monolithic data model. We meticulously examined each collection within the MongoDB database to identify clusters of domains and transaction boundaries.

Entities such as Users, Products, and Orders emerged as the central elements of bounded contexts. Relationships between these fundamental entities became candidates for service decomposition. For instance, we observed that Orders contained foreign keys to Users for customer information and Products to represent purchased items.

To gain a deeper understanding of these interdependencies, we printed example documents to visualize associated fields. An intriguing discovery was that legacy code duplicated data that now belonged to separate business capabilities. For instance, shipping addresses needlessly repeated user profiles instead of storing a lightweight reference.

This analysis of relationships revealed problematic tight coupling between modules, resulting in cascading updates. The normalization of redundant data removed barriers to independent development for user profiles and shipping namespaces.

We employed database tools to explore the intricate network of connections. Utilizing MongoDB Compass, we diagrammed relationships through $lookup pipelines and executed aggregate queries to tally references between entities. This revealed pivotal breakpoints for splitting logic into coherent services.

These relationships informed the demarcation of domain boundaries and ensured that services exposed clean, well-defined interfaces. Such well-defined contracts empowered autonomous teams to incrementally develop and deploy modules as micro frontends without obstructing one another.

### Identifying Transactional Boundaries

Beyond just relationships, we delved into the transactions embedded in the existing codebase to comprehend the flow of business processes. This exercise helped pinpoint where data modifications needed to be confined within individual services to uphold data consistency and integrity.

For instance, when dealing with order processing, we recognized that any updates related to the order itself, associated payments, inventory levels, and shipment notifications had to occur within a single service, in a transactional manner. This realization played a pivotal role in shaping the boundary of our Order Management service.

A thorough analysis of both relationships and transactions yielded invaluable insights that guided the refactoring of the data model and logic into independently deployable microservices, each characterized by clearly defined interfaces.

## Refactoring for Microservices

### Normalizing Data Schemas

To accommodate independent services that may deploy to different data stores if necessary, we embarked on the normalization of schemas. The goal was to eliminate redundancy and include only the minimal data required by each service.

For example, the original Orders schema contained the entire User object. We streamlined this by creating a lightweight reference:

```
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, we extracted product details from Orders and placed them into their own collections. This approach allowed these entities to evolve independently over time.

### Applying Domain-Driven Design

We leveraged the bounded contexts concept from Domain-Driven Design to logically segregate services, such as Order Fulfillment versus User Profiles. Interfaces served as a vital abstraction for data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands underwent refactoring to align with the new architectural paradigm. In the past, services directly accessed data using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility, allowing us to migrate databases without necessitating changes to consumer code.

## Deploying Microservices

### Autonomous Scaling

In the realm of microservices, autoscaling is performed at the individual service level rather than the application-wide level. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests outlined scaling policies based on

 CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers offered reserve capacity and elastic overflow handling. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers like Nginx and Traefik skillfully routed traffic to scaled replica sets. This optimized resource utilization, bolstering throughput and reducing costs in the process.

### Enforcing Resilience

Our arsenal of resiliency techniques encompassed retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as shields against cascading failures, while the Platform service took charge of transient error policies for dependent services.

Both homemade and open-source solutions, including Polly, Hystrix, and Resilience4j, stood as guardians against potential mishaps. Centralized logging via Elasticsearch played a pivotal role in tracing errors across the distributed applications.

### Implementing Reliability

Incorporating reliability into microservices mandated the incorporation of various techniques aimed at averting single points of failure. We prioritized automated responses to transient errors and designed measures to tackle overload scenarios.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully handle faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was employed to prevent service flooding during times of stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were employed to abort lengthy calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These techniques guaranteed consistent responses and thwarted cascading failures across the spectrum of services.

## In Conclusion

The migration of our monolithic ERP system into microservices has been an invaluable learning experience. It transcends the realm of a mere technical migration, signifying an organizational metamorphosis that equips our client to better serve their ever-evolving customer base.

By dismantling tightly coupled layers and instituting clear domain boundaries, our development team has unlocked a new level of agility. Features can now be developed and deployed independently, driven by business priorities rather than architectural constraints. This capacity for rapid experimentation and refinement positions the application to remain responsive to dynamic market demands.

Simultaneously, our operations team now enjoys full visibility and control over each system component. Anomalous behavior can be promptly detected through improved monitoring of discrete services. Scaling and failover mechanisms have transitioned from manual interventions to automated processes, fostering enhanced resilience that will serve as a reliable foundation for our client's continued growth.

While the benefits of microservices are unequivocal, embarking on such a migration is not without its challenges. By dedicating time to meticulous relationship analysis, interface definition, and abstraction introduction—rather than resorting to a crude 'rip and replace' approach—we have cultivated a flexible architecture capable of evolving in tandem with customer needs.

Above all, I am profoundly grateful for the opportunity this project has afforded us, allowing us to collaborate closely with our client on their digital transformation journey. Sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses, are unquestionably worth the

 endeavor.

## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
