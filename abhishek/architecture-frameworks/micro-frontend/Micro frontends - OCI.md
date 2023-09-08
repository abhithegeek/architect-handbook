Most of us are familiar with the [microservices](https://martinfowler.com/articles/microservices.html) architecture and the benefits it brings to building complex backend systems. However, as much as microservices have revolutionized how we build services, the story on how to build responsive and cohesive user interfaces at scale isn’t as clear. Today, multiple design philosophies and technologies exist to build and scale complex UIs, each with their own pros and cons.

This post talks about our journey building the [Oracle Cloud Infrastructure (OCI) Console](https://cloud.oracle.com/), a unified, web-based management portal for OCI. The Console enables OCI customers to manage and monitor their cloud resources and subscriptions. We talk about how we scaled the Console to support millions of page views. We also talk about how hundreds of teams work together to build the Console.

## Scaling the Console

We started with trying to answer the question, “What rendering and delivery architecture can meet the scalability and flexibility required for the Oracle Cloud Console?”

At the time, [server-side rendering](https://web.dev/rendering-on-the-web/) (SSR) and a [dedicated application tier](https://en.wikipedia.org/wiki/Multitier_architecture) (UI service) were the most common design patterns for developing complex UIs. In these design patterns, a translation layer sits between the UI and the backend services. This layer stores the data model and makes API calls on behalf of the user. This intermediate service can also prerender the view for the client browser to display. The main advantage of SSR is its ability to load pages faster, even on slow devices or networks. However, the trade-off of a UI service that aggregates data is that it adds complexity and limits our rate of innovation. This aggregation layer becomes a bottleneck for scaling UIs. With these points in mind, we decided to go with a fully client-side, API-centric solution for rendering the Console.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT07A543741E45489E9E2D13E7175A82A0/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 1. Comparison of different rendering options for the Console. The Console uses the client-side rendering approach.</em></sup>

## Client-side rendering (CSR)

We decided to treat the Console as another client of OCI’s backend [REST APIs](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/usingapi.htm). The Console doesn’t depend on secret or hidden APIs but instead uses the same APIs available to our customers through the [command-line interface (CLI) and software developer kits (SDKs)](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdks.htm). As a result, we’re no longer bottlenecked in our innovation by an aggregation layer, and we can extend the Console at the same rate that [OCI services](https://docs.oracle.com/en-us/iaas/Content/home.htm) are added or updated.

With CSR, we encapsulate the business logic in Javascript and rely on the client (user’s browser) to do most of the heavy lifting of rendering. With this architecture, the Console is just a collection of static assets, such as Javascript, HTML, and CSS, that can be served from any file server or [content delivery network](https://www.akamai.com/our-thinking/cdn/what-is-a-cdn) (CDN). This model makes scaling the Console trivial.

This approach gave us flexibility in how we build and operate the Console. CSR has the following advantages:

-   Scalability: Console usage scales with the size of our distribution network. The Console gets millions of page views globally every month.
    
-   Maintainability: Because the data model lives with the view, we can easily keep them in sync as the model evolves.
    
-   Simplified testing: Console developers can build their feature branch and upload test assets to [OCI Object Storage](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorageoverview.htm). You can now share and run these assets locally with a single link, effectively giving developers unlimited test environments to validate multiple versions of the Console being developed in parallel.
    
-   High availability: We can cheaply build multiple redundant stacks (multiple CDNs or multiple high availability failover zones) to serve Console assets and ensure that the Console almost never goes down.
    
-   Business agility: We can rapidly bootstrap and deploy multiple instances of the Console, for example, to support [new regions](https://www.oracle.com/cloud/cloud-regions/), [dedicated regions](https://www.oracle.com/cloud/cloud-at-customer/), or even internet-disconnected [national security regions](https://www.oracle.com/industries/government/govcloud/classified/).
    

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT91E80A512BD9463AB7C42C0712CC124B/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 2. Delivery mechanism for the Console. Console assets are deployed to a CDN and rendered in the user’s browser.</em></sup>

## Console performance

By moving all rendering to the browser, we give up the performance benefits that are naturally conferred by using server-side rendering. Despite this fact, the Console has a load time of only about 2 seconds from a cold cache, according to [First Meaningful Paint](https://developer.mozilla.org/en-US/docs/Glossary/first_meaningful_paint#:~:text=First%20Meaningful%20Paint%20(FMP)%20is,differences%20in%20the%20page%20load.). The Console’s performance is primarily affected by the following variables:

-   The size of our static assets (especially Javascript): Large files take longer for a browser to download and parse, increasing rendering time.
    
-   The number and sequence of REST API calls to OCI services, which determines how long it takes the rendered page to show meaningful data
    

To reduce the impact of these variables and consequently improve the Console’s performance, we used a combination of the following techniques:

-   Telemetry: We constantly collect and analyze performance metrics across hundreds of dimensions and monitor for regressions.
    
-   Tooling: We analyze our builds and block check-ins that impact Console performance, such as detect increased asset size and duplicate dependencies. We also use build optimizations, such as [tree shaking](https://webpack.js.org/guides/tree-shaking/) and [code splitting](https://webpack.js.org/guides/code-splitting/) to further reduce asset sizes.
    
-   Deferred rendering: We minimize unnecessary data transfer over the network. We [lazy load](https://webpack.js.org/guides/lazy-loading/) assets and download only what’s required to render the current screen.
    
-   Progressive rendering: We improve perceived load times by immediately rendering smaller portions of the page as data from API calls becomes available.
    
-   API parallelization: We parallelize and [multiplex](https://web.dev/performance-http2/) our API calls using HTTP/2 to reduce latency and increase concurrency.
    
-   Asset caching: We use a geographically distributed CDN to cache and bring assets closer to our global customers. We also use browser caching to further reduce impact of asset downloads.
    
-   Data caching: We use [IndexedDB](https://github.com/jakearchibald/idb-keyval) to cache API responses in browser storage and apply a leaderless replication mechanism to keep the cache in sync with backends.
    

We also wanted to avoid situation in which a few senior developers had all the context and became bottlenecks. Instead, we invested in technology solutions: Tools that make it easy for anyone to write performant code. For example, the Console has almost two dozen dependencies during initialization: Authentication, localization, user preferences, and feature toggles, to name a few. Manually keeping track of these dependencies and their order is the perfect setup to introduce latent bugs and performance issues.

To solve this problem, we built an orchestration library that allows Console developers to define initialization dependencies declaratively. These abstractions enable developers to focus on the UI and customer experience, while letting the frameworks take care of implementation details.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONTA3DDF3C1CE334C7894B4DCB0EF65AD7C/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 3. The orchestration library guarantees performance and efficiency during Console initialization. It manages dependency initialization by constructing an optimal <a href="https://en.wikipedia.org/wiki/Directed_acyclic_graph">directed acyclic graph</a> (DAG) of the initialization sequence. In addition to simplifying dependency management, the DAG constructed by the library can be statically inspected and debugged for performance issues.</em></sup>

## Microfrontends

Now that we had a technology solution to scaling, we built Console to support the first wave of OCI services: Compute, networking, storage, and databases. As was the norm for UI development at the time, we built the Console as a monolith, a single-codebase, contained UX that was responsible for everything from managing [instances](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/launchinginstance.htm) to creating [virtual networks](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/overview.htm) or [databases](https://www.oracle.com/autonomous-database/). A single, central Console team built the UX for all OCI services.

However, as OCI matured, it bootstrapped new services and released new features faster than the Console team could build UX for them. The Console code was also becoming larger and harder to manage, while the Console team was spending increasing cycles coordinating efforts with dozens of service teams.

To solve this human scaling problem, we turned to a federated design pattern called [microfrontends](https://micro-frontends.org/) as the design solution for sustaining the hyper growth of OCI. The [microfrontend](https://martinfowler.com/articles/micro-frontends.html) pattern enables hundreds of teams to work simultaneously on a large and complex product.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT610AE01DCF0A4D34BD12BF4D37637F04/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 4. Microfrontend development model. Service teams own both the backend and the corresponding UI. Each UI plugin can be built, deployed, and managed independently.</em></sup>

Microfrontends are a [federated](https://en.wikipedia.org/wiki/Federated_architecture) web development technique that’s a logical extension of the popular microservices architecture. Instead of a single horizontal team developing all UI, individual service teams focus on their verticals (business domain) and deliver end-to-end experiences, including user interfaces tailored to their specific needs, and the backend layers. Service teams develop miniature UI applications called plugins. Teams independently develop their plugins and release them at a cadence that suits their customers’ needs.

Microfrontends have the following advantages over monoliths, which allow the Console to continue to scale and evolve:

-   Smaller and more focused codebases
    
-   Decoupled, autonomous teams
    
-   Operational flexibility and speed
    

Figure 5 shows the high-level architecture of a microfrontend-based Console. The Console has three major components: Application, plugins, and runtime. The Console becomes a UI composition system which integrates plugins built by hundreds of teams into a [single-page web application](https://en.wikipedia.org/wiki/Single-page_application) in the browser. This composed application appears to the end user like it was built by a single team.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONTF2E5A54342F44766BFECE0C74A812978/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 5. Architecture diagram of the microfrontend-based Console, showing the three major components: Application, plugins, and runtime.</em></sup>

### Application

The application is the entry point of Console. The application is responsible for authenticating the user and retrieving authentication tokens. It also renders the chrome (Header, footer, navigation menu) and the homepage. The application initializes all required modules and maintains state globally in a context object. It also loads and renders the appropriate plugin based user interaction.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT140022CA224149779FFE7B875B3EF184/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 6. Anatomy of the Console application</em></sup>

### Plugins

A plugin is a mini application developed by a service team to represent the UX and business logic of their service. Plugins can’t run on their own and must run within the Console application. New plugins onboard to the Console by adding metadata about their plugin into a plugin registry. Plugin metadata includes information such as the plugin name, asset url, and route mapping.

![Figure 7. A Console plugin page. The application loads the plugin in an iframe.](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT94719E672A0648DA90A65FBA68117F63/native?cb=_cache_9257&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 7. A Console plugin page. The application loads the plugin in an iframe.</em></sup>

Plugins are sandboxed to ensure that a misbehaving plugin doesn’t impact the rest of the Console. To achieve sandboxing, we had a couple different options to choose from: [iframes](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe) and [web components](https://www.webcomponents.org/introduction). At the time, web components didn’t have widespread browser support, nor did it satisfy the Console’s requirements for sandboxing. We chose iframes as the simplest approach to compose microfrontends in the browser at runtime.

iframes have traditionally been much maligned. Previously, browsers had poor support and poor performance for iframes. However, modern browsers are much more performant and provide fine-grained control over iframes. Console successfully uses iframes to gain the following benefits of sandboxing:

-   Privacy boundary: User details and authentication material (id and security tokens) are never shared with plugins.
    
-   Security boundary: Untrusted code doesn’t have access to other plugins, the document object model (DOM), browser storage, or [globals](https://developer.mozilla.org/en-US/docs/Glossary/Global_object#:~:text=A%20global%20object%20is%20an,members%20of%20the%20global%20object.).
    
-   Event boundary: DOM events don’t interfere with other Console components.
    
-   Style boundary: CSS styles are isolated and don’t rely on developer hygiene to avoid namespace conflicts.
    
-   Error boundary: The Console can gracefully recover from errors. Errors or bugs don’t crash the Console application and are scoped to plugins.
    
-   Performance boundary: Modern browsers run iframes in their own process ([Out-of-process iframes](https://www.chromium.org/developers/design-documents/oop-iframes/#:~:text=OOPIFs%20were%20motivated%20by%20security,%3E%20tag%20in%20Chrome%20Apps).)). Performance issues are scoped to plugins and don’t impact the Console application.
    
-   Technology boundary: The Console can simultaneously host plugins created with different UI technologies. While the Console team provides a recommended UI technology stack based on [React](https://reactjs.org/), service teams are free to choose their own stack if required.
    

### Runtime

The runtime is the heart of Console’s microfrontend strategy. While iframes offer many great benefits, the easy isolation that they provide makes them less flexible. Building integrations between different parts of the Console can be challenging. To solve this problem, we built the Console runtime: A set of libraries that provide a highly opinionated, batteries-included approach to microfrontends. The runtime has three major functions: Communication channel, modules, and plugin manager.

The communication channel allows bidirectional message passing over the iframe boundary to enable different parts of the Console to communicate with each other asynchronously. To a plugin, the runtime provides this functionality transparently using standard Javascript [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) and the async or await syntax. We use the [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) webAPI to do the actual message passing over the iframe boundary. The runtime sets up message queues to which messages are delivered. Various consumers subscribe to the message queues for topics relevant to them, enabling an event-driven model where loosely coupled components communicate with well-defined contracts.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONTD55A817FED344AF1BE35E5F82E1A4D9E/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 8. The communication channel allows components to send messages across the iframe boundary using standard Javascript Promises.</em></sup>

Modules are abstractions that offer functionality to plugins, without the plugins having to know the implementation details. Modules have two parts, the plugin client and the application listener, which use the communication channel. A plugin uses the client to send a message to the corresponding listener. In turn, the listener processes and responds to the message.

  ![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONT1C59AFE4ACFF43108FB39A392F9A1D2D/Medium?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)  
<sup><em>Figure 9. Modules have a client-listener architecture and use the communication channel for passing messages.</em></sup>

Each module is responsible for a single piece of function. The following list shows a few modules and their functions the runtime supports:

-   Logging, metrics, and analytics: Allow plugins to emit telemetry
    
-   Request signing: Allows plugins to sign backend requests without exposing user authentication material to plugins
    
-   Error handling: Allow plugins to notify the Console application of any unhandled Javascript errors
    
-   Routing: Allow plugins to control browser URL and browser history
    
-   Feature toggles: Allow plugins to turn features on or off
    
-   A-B testing: Allow plugins to roll out new features as experiments
    
-   Personalization: Allows plugins to persistently store user preferences
    

The plugin manager is responsible for managing a plugin’s lifecycle. When a user expresses intent to navigate to a plugin path, the application invokes the plugin manager to load the plugin into the desired DOM container. The plugin manger looks up the plugin from the registry, downloads its assets (HTML or Javascript) and creates a new iframe hosting the plugin. The plugin manager also creates the required communication channels and sets up a direct line to monitor the plugin. When a user navigates away or a plugin is no longer visible, plugin manager suspends or deletes the iframe.

## The future of the Console

Client-side rendering and the microfrontend patterns have been incredibly successful for the Console. They have allowed us to build a stable, performant, and operationally mature UX platform used by millions of OCI customers. Rich tooling, opinionated frameworks and an easy-to-use, extensible platform has allowed hundreds of service teams to onboard their UX to Console. As OCI continues to scale, and as Console has matured as a product, our goals have expanded in the following areas:

-   Consistency: We want to make it easier for plugin authors to build and maintain new plugins, while increasing UX consistency among plugins. For example, plugin accessibility is time-consuming and hard for service teams to get right. To solve this issue, we’re building a configurable rendering engine for the Console to handle the heavy lifting for common UIs and experiences across the Console, letting plugin developers to focus on what makes their plugin unique.
    
-   Observability: We want greater insight into how service teams are using the Console platform and how our customers are using the Console. We’re beefing up the communication channel into a fully event-driven runtime architecture to offer unprecedented levels of visibility into plugins and users.
    
-   Flexibility: We want the Console to adapt to you, instead of the other way around. We’re working on building an intent-driven Console that prioritizes and responds to your intent and personas, such as database admin, finance specialist, and DevOps. With an intent-driven Console, we’re moving from “a plugin is the unit of federation” to “an intent is the unit of federation.” As a result, the Console can offer flexible and dynamic UX that you can tailor to your needs.
    

To support these goals, we’re constantly prototyping the latest in web development technologies and standards. Our beloved iframe-based plugins have served us well and we intend to keep them around, but technologies such as [web components](https://developer.mozilla.org/en-US/docs/Web/Web_Components), [custom renderers](https://github.com/tc39/proposal-ses), [shadow realms](https://github.com/tc39/proposal-shadowrealm), and [Secure ECMAScript](https://github.com/tc39/proposal-ses) offer more flexible, standards-based alternatives to iframe-based microfrontends, while maintaining the sandboxing features they provide. Stay tuned for updates!

## Want to know more?

Engineers at OCI are constantly innovating and iterating to ensure excellent products and services are delivered to customers. This blog series highlights the new projects, challenges, and problem-solving OCI engineers are facing in the journey to deliver superior cloud products. We have more of these OCI engineering team highlights as part of this Behind the Scenes with OCI Engineering series, featuring talented engineers working across Oracle Cloud Infrastructure.

See more of our [Behind the Scenes with OCI Engineering](https://blogs.oracle.com/cloud-infrastructure/category/oci-behind-the-scenes) posts.

![](https://blogs.oracle.com/content/published/api/v1.1/assets/CONTBD314C2D6152422F84A24C6F2728A202/Thumbnail?cb=_cache_9257&format=jpg&channelToken=f7814d202b7d468686f50574164024ec)

#### Abishek Murali Mohan

##### Consulting Member of Technical Staff, OCI Console Team