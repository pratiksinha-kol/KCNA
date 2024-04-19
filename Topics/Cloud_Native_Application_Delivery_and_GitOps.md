## Cloud Native Application Delivery and GitOps

As part of your KCNA study, these are the main points of consideration for the Cloud Native Application Delivery and GitOps section -

1. Be aware of Argo and how this software integrates with the GitOps lifecycle

2. Knowledge that Argo can be further utilised through the use of Argo Workflows

3. Have an understanding and awareness of Flux, an Argo alternative

4. Have an understanding of the GitOps toolkit which Flux utilises

** ** 

## Cloud Native Application Delivery and GitOps - Further Study

In our lesson, we provided an introduction to Argo CD. To extend your knowledge further and for the purposes of the KCNA examination, we recommend familiarising yourself with the Argo CD Workflows offering and an understanding that workflows are available to enhance the capabilities of Argo CD in managing complex deployment processes.

It is also important from an exam perspective to have an awareness of Flux, an ArgoCD alternative and in particular, be aware that Flux may be a more appropriate solution for synchronised cluster changes. This is explained and detailed below. From an examination viewpoint please pay particular attention to **Configuration Sync**:

## Argo CD Workflows
For more information, visit the Argo Workflows documentation at https://argoproj.github.io/workflows/

Argo CD workflows are central to managing complex deployment processes. These workflows allow for the orchestration of multi-step deployment processes, integrating with the Argo Workflows project for advanced automation capabilities.

Key aspects include:

- **Sequential and Parallel Execution:** Workflows can be configured to run tasks either in sequence or parallel, providing control over the deployment process.

- **Integration with Argo Workflows:** This integration allows for the creation of complex pipelines combining CI (Continuous Integration) and CD (Continuous Deployment) tasks in a unified workflow.

- **Customisable Pipelines:** Users can create custom pipelines to match their specific deployment requirements, such as incorporating testing stages, approval processes, or integration with external tools.

## Introduction to Flux
While our lesson primarily focused on Argo CD for implementing GitOps principles, it's beneficial to broaden your awareness of options in this space by exploring Flux, another prominent GitOps continuous delivery solution. Flux is an open-source project maintained by the CNCF, with detailed documentation available at [Flux's official website](https://fluxcd.io/).

### Comparison Between Argo CD and Flux
Both Argo CD and Flux are powerful tools for GitOps, but they have distinct features and architectural approaches.

- **Architecture and Operation:** Argo CD follows a more centralised approach, with a single point of control for deployment. In contrast, Flux adopts a more decentralised stance, where each Flux instance operates independently, making it potentially more scalable in large systems.

- **Configuration Sync:** Both Argo CD and Flux can continuously monitor a Git repository and can automatically apply changes. In the video demonstration for Argo CD you’ll note that we optionally selected the synchronise option when setting up our app. It would have been possible if we wished to toggle this and to then synchronise manually later if we wished.

Flux however has always traditionally used a pull-based approach and was one of the first tools in the Kubernetes ecosystem to introduce this capability as one of it’s core features. It will **automatically** synchronise changes.

Flux’s original design was strongly centered around it’s synchronisation process and this approach has made Flux synonymous with the GitOps approach of continous synchronisation between the repository and cluster. For this reason, many would consider Flux the **“textbook” choice for continous synchronisation**, even though both Flux and ArgoCD share this capability.

- **Customisation and Extensibility:** Flux is built on the GitOps Toolkit, a set of composable APIs and specialised tools, allowing for more customisation and extensibility. This modular approach lets you tailor Flux to fit more complex scenarios.

- **Ecosystem and Community:** Both tools have strong communities and are part of the CNCF landscape. However, their ecosystems differ slightly, with Flux being more aligned with the broader CNCF ecosystem, potentially offering better integration with other CNCF projects.

## Flux and the GitOps Toolkit
Flux's use of the GitOps Toolkit is a significant aspect that sets it apart from other similar tools. The GitOps Toolkit is a collection of APIs and controllers that can be combined to create a complete GitOps workflow. These components include:

- **Source Controller:** Manages the acquisition of source code from repositories like Git.

- **Kustomize Controller:** Applies Kubernetes manifests using Kustomize, a tool for customising Kubernetes configurations.

- **Helm Controller:** Allows for the declarative management of Helm chart releases.

- **Notification Controller:** Handles alerts and notifications for the GitOps workflow.

These components provide a flexible and powerful way to implement GitOps, allowing for more granular control and customisation of the continuous delivery process.

While Argo CD is an excellent GitOps tool that we've explored in our lesson, understanding Flux and its use of the GitOps Toolkit can provide you with a more comprehensive view of the GitOps landscape. Both tools have their strengths, and knowing when to use one over the other can be crucial in certain deployment scenarios. For those interested in delving deeper into Flux, its documentation is a great starting point.