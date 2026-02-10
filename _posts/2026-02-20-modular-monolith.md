---
title: Scalable Modular Monolith
categories: [arcitecture]
tags: [architecture,monolith,modular]
published: false
---
Microservices are a popular architectural style for building scalable and maintainable applications. However, they also introduce complexity and overhead in terms. Microservices are great for large application with many teams working on different parts of the application, but for smaller applications or teams, a modular monolith can be a better choice. 

Below is a comparison between microservices and modular monoliths for small teams and applications.

| Aspect          | Microservices                                                                                      | Modular Monolith                                                         |
|-----------------|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| **Deployment** | Multiple services to manage, version, and deploy independently                                      | Single artifact; simpler deployment process                              |
| **Maintainability in Small Teams)** | High overhead; teams need expertise across infrastructure and DevOps           | Better fit; easier to understand and maintain single codebase            |
| **API Spec Coordination** | Requires formal API contracts and documentation; changes need careful versioning         | Direct code coordination; changes are immediately visible across modules |
| **Versioning Between Dependencies** | Complex vesioning; services must support multiple API versions                   | Easier to refactor; all modules updated together in same release         |

### Modular Monolith
When starting a new project, it can be beneficial to start with a modular monolith architecture. This allows you to build a single application that is organized into modules, which can be developed and tested independently. As the application grows, you can then refactor it into microservices as needed.

### Horizontal Scale
The make the monolith horizontally scalable make sure it is stateless. This means that the application should not rely on any in-memory state, and should instead use a database or other external storage for all stateful data. This allows you to easily scale the application horizontally by adding more instances, and also makes it easier to refactor into microservices later on.

### Feature Flags
All functionality is one artefact which makes deployments are simpler. You can also use feature flags to enable or disable features as needed, which allows you to deploy new features without affecting the entire application. Environment variables can be used to control the features.

## Modes of operation
Similar to feature flags, you can also have different modes of operation for the application. For example, use an environment variable to run application in db migration mode. Where
the application will only run database migrations and then exit. This can be useful for performing maintenance tasks or running the application in a maintenance mode

### Coordination between dependencies
Module dependencies are easier to manage in a monolith, as all modules are part of the same codebase. This allows you to easily coordinate changes across modules, and also makes it easier to refactor code as needed. In a microservices architecture, coordinating changes across services can be more complex, as each service may have its own codebase and deployment pipeline.


### Below is package structure for a modular monolith application. Each module is organized into its own package in java, and the main application is responsible for coordinating the modules.

```txt
com.example.app
├── main
│   ├── Application.java
│   └── config
│       └── AppConfig.java
├── module1
│   ├── Module1Service.java
│   └── Module1Controller.java
├── module2
│   ├── Module2Service.java
│   └── Module2Controller.java
└── module3
    ├── Module3Service.java
    └── Module3Controller.java
```

The diagram belows how a stateless modular monolith can be horizontally scaled by adding more instances of the application. Its simple 3 tier architecture with a load balancer in front of multiple instances of the application, which can be scaled up or down as needed. State is stored in a shared database, cache or other external storage (nfs), which allows the application to be stateless and easily scalable.
```txt
          +------------------+
          |   Load Balancer  |
          +------------------+
                   |
        +----------------------+
        |                      |
+------------------+    +------------------+
|   Application    |    |   Application    |
|     Instance 1   |    |     Instance 2   |
+------------------+    +------------------+
    |                            |
    +---+-------+-------+--------+
                |            |
   +----------+  
   | Database | 
   +----------+
``` 