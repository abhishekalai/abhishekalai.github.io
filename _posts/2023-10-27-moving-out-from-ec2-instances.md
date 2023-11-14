---
layout: post
title:  "Moving out from EC2 instances or Linux VMs for scale"
date:   2023-10-27 15:45:32 +0530
categories: aws, ec2, software-development, deployment, applications
---

Level: `intermediate`

Pre-requisites: AWS, EC2 Instances, Linux servers, Basics of Networking & Routing, Traffic estimation

### Overview
A lot of developers or organizations (especially startups) start with hosting their web applications on EC2 instances (in AWS) or Virtual Machine (VM) instances (in Google Cloud Platform) which are not scalable by default. Once they are out of the budgetary restrictions or the PoC (Proof of Concept) stage, it's time to deploy the application using a scalable service to handle all the new users and traffic who come from the bigger marketing campaigns and expansions. In this write-up, we go through some basic strategies to determine to choose the best deployment option for our applications.

### Terminology
In this write-up, the term Web Application implies application that can be served over web browser based protocols, which includes **HTTP-REST APIs** (also known as **Backend APIs**) and client applications. The client applications include the UI based HTML applications rendered in the browser and are also known as **Frontend Applications**.

Let's begin.

### Frontend Applications
Frontend applications traditionally were served using web servers like [Apache][apache_web_server_project] or [Nginx][nginx_web_server_project]. These applications usually had a `index.html` file at the root level of the project which loaded all other scripts and assets using HTML tags like `style`, `script`, `img` etc.

Modern day Single Page Applications (SPAs) built with frameworks like [ReactJS][react_homepage], [Angular][angular_homepage], [Vue][vue_homepage] etc. give a better Developer Experience and code maintainability and compile the application to a similar structure that of the traditional web apps.

SPAs can be hosted in a very cost-efficient manner using CDN services. Some popular offerings in this space are:
- AWS Cloudfront backed with AWS S3 storage
- Cloudflare static apps
- Netlify
- Vercel
- Google Firebase hosting

**CDNs are priced to serve large amounts of traffic and hence become cost-effective.** Hosting a web app on a server will give following benefits:
- **Better SEO**, as crawlers have to parse millions of web pages, they may or may not wait for the entire bundle to be downloaded in case of an SPA
- **Faster response times** as the user sees a render immediately as only the page they've requested is sent to them
- **Caching** of static assets will ensure faster page loads for your sites / apps.

![apps-on-cdn-artwork](/assets/apps-on-cdn.png)

All of the major frontend frameworks/libraries have popular Server-side rendering (SSR) packages & support available. All major cloud providers have offerings that allow you to host SSR apps. Some examples:
- GCP Cloud Run
- AWS Amplify
- Vercel
- Heroku

If you are using a popular library, there is a high possibility that your Cloud Provider has a deployment template available for it. These templates often help you get started with minimal configuration and provide auto-scaling as a standard feature. It is important to note that these services provide abstraction on their existing suite of services and remove a lot of configuration hassle for the user, and hence, granular configurations like network, storage, routing etc. will not be available.


### Backend Applications
Backend or the APIs are responsible for handling logic & data operations for your applications. And, in the competition among the Cloud Service Providers (CSPs), it's the developers that win.

All of the big players have a plethora of offerings that will help you host your applications with ease and handle the scale that you are looking for. Here are some possible combinations of deployment architectures to consider:

- **Serverless Computing** enables the developers to just upload their code artifact and the rest is taken care of by the CSP. This includes the auto-scaling part.  AWS Lambda is the current market leader in this space and provides native integrations with almost all of the offerings in AWS bouquet. Google Cloud Functions, MS Azure Functions are the next big options.

    ![serverless offerings](/assets/serverless_offerings.png)

- **Auto-scaled VMs** consists of deploying your applications in linux VMs behind a load balancer or a gateway/proxy that automatically balances the traffic between the [horizontally scaled][horizontal_scaling] VM group. This generally requires you to define auto-scaling rules bases which your application is scaled by your CSP.
- **Container-based architectures** are available in both variants - **Managed** and **Self-managed**.

    A managed offering will ask you to upload or point to your application's [docker image][docker_images] and ask for minimal configuration like Environment variables and Port numbers, min. or max. number of containers for scaling up / down etc. While the configuration options look minimal, there is enough flexibility to scale your application. GCP Cloud Run and AWS Elastic Container Service (ECS) Fargate are some of the prominent offerings in this space.

    A self-managed offering will additionally allow you more customizations, like choosing a [Virtual Network][vnet_definition] for deployment, traffic routing, custom CIDR blocks and IP Address definitions, load balancing, granular auto-scaling rules etc.
- **Microservices architecture** is a common pattern in bigger businesses and requires a cultural shift to get it right. Each microservice is independent and by principle, keeps on working even if some other service fails. Most common implementation for this pattern consists of services getting deployed as containers and being orchestrated by a tool like [Kubernetes][k8s].


### Conclusion
There are multiple options and patterns available to scale a web application on cloud. However, while finalizing the service provider and architecture pattern, it is important to consider factors like complexity, maintenance overhead and operational costs. We'll look at options for databases in upcoming posts.


<!-- Links -->
[apache_web_server_project]: https://httpd.apache.org/
[nginx_web_server_project]: https://nginx.org/en/
[react_homepage]: https://react.dev
[angular_homepage]: https://angular.io
[vue_homepage]: https://vuejs.org/
[horizontal_scaling]: https://wa.aws.amazon.com/wat.concept.horizontal-scaling.en.html
[docker_images]: https://docs.docker.com/trusted-content/official-images/
[vnet_definition]: https://www.fortinet.com/resources/cyberglossary/virtual-cloud-network
[k8s]: https://kubernetes.io/