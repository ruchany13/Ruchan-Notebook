# What is an Image Registry?

Imagine you're a chef preparing meals from recipes. Every dish you make comes from a well-defined set of ingredients and instructions. In the software world, container images are like those recipes – they define everything your application needs to run. But once you've created an image, where do you store it so others (or you, on another server) can use it?

That's where **image registries** come in.

An **image registry** is a centralized system that stores and manages **container images**. These images include all the code, dependencies, and configuration required to run an application. Developers, DevOps teams, or automated systems use registries to **push** (upload) and **pull** (download) images during software development and deployment.

But there's another term we should clarify early: **repository**.

### Registry vs Repository
- A **registry** is the central service that stores images.
- A **repository** is a collection of related images under the same name but different versions (tags). Think of it like a folder in your computer containing different versions of a file.

> 🧠 For example, a registry might host a repository named `myapp`, which contains images tagged as `v1`, `v2`, `latest`, etc.

### Why Do We Need Registries?
If you’re working alone on a single machine, you might not need a registry. But in most real-world scenarios:
- You have **multiple environments** (development, staging, production)
- You're working in **teams**, often distributed
- You want to **version**, **distribute**, and **reuse** your images efficiently

For example, suppose a developer builds an application container on their laptop. To deploy it to the cloud or make it available to a colleague, they need a place to share it – a registry.

Registries also allow for better control and automation in CI/CD pipelines. Once an image is built, it’s pushed to a registry. Later, deployment systems pull from that registry to deploy the app.

Registries can be public or private:
- **Public**: Anyone can pull images. Example: Docker Hub
- **Private**: Access restricted to specific users or systems. Example: self-hosted Docker registry, Harbor, GitLab Container Registry

---

## Docker Image Registry: A Popular Implementation

One of the most widely used image registry systems is the Docker image registry, which adheres to the open container registry specification. It powers platforms like Docker Hub and can also be self-hosted using tools like the official Docker Registry image or third-party tools such as Harbor.

Docker Hub hosts:

- ✅ Official base images (e.g., `nginx`, `ubuntu`, `python`)
- 🧰 Community-contributed images
- 🔐 Private repositories with access control

Docker Hub is the default public registry used when you run commands like `docker pull ubuntu` or `docker push myapp`. It hosts millions of open-source images and also allows private repositories with authentication.

---

## Using Docker Hub: Basic Workflow

To interact with Docker Hub, you typically follow these steps:

### Step 1: Create a Docker Hub Account

- Go to [hub.docker.com](https://hub.docker.com/)
- Sign up for a free account

### Step 2: Pull a Public Image

```bash title="Image Pull from Docker Hub"
docker pull nginx
```

This command downloads the official `nginx` image from Docker Hub to your local machine.

### Step 3: Tag the Image for Your Own Repository

```bash title="Image Tag"
docker tag nginx yourusername/nginx-demo:latest
```

- `yourusername` should match your Docker Hub username
- `nginx-demo` is the name of the repository you're creating
- `:latest` is the tag (you can use versions like `:v1.0` as well)

> 🔖 **Tagging** is how Docker identifies versions of an image inside a repository. The format is: `repository:tag`

💡 **Why use your username in the tag?**
Because Docker Hub organizes repositories by username, using your Docker ID as a namespace avoids conflicts and tells Docker where to push the image.

### Step 4: Push the Image to Your Repository

Firstly, login docker hub with your credentials
```bash title="Docker Login"
docker login
# Enter Docker Hub credentials
docker push yourusername/nginx-demo:latest
```

Push image
```bash title="Image Push"
docker push yourusername/nginx-demo:latest
```

Now the image is published to your Docker Hub account and accessible from anywhere with:

```bash title="Image Pullé
docker pull yourusername/nginx-demo:latest
```

---

## Public vs Private Registries

A **public registry** allows anyone to pull container images without authentication. For example, **Docker Hub** hosts widely used public images like `ubuntu`, `nginx`, or `node`, which can be pulled directly without credentials.

A **private registry**, on the other hand, restricts access to authorized users. These registries can be hosted by cloud providers — such as **AWS ECR**, **GitLab Container Registry**, or **Azure ACR** — or run on your own infrastructure as a **self-hosted solution**.

When to use which?

- ✅ Use **public registries** for open-source projects, community sharing, and base images.
- 🔒 Use **private registries** for proprietary software, internal development, and secure deployments.

> 💡 Self-hosting your registry gives you complete control over who can access your images, how they are stored, and how they are managed — a topic we’ll explore next.

---

## Why Use a Self-Hosted Private Registry?

Running a self-hosted image registry gives you full control over how your images are stored, accessed, and managed. Here are the main reasons teams choose this route:

### ✅ Advantages

- **Full Control**: You manage user access, storage paths, networking, and backups.
- **No Rate Limits**: Unlike Docker Hub, which limits anonymous and free-tier access, your private registry won't throttle requests.
- **Security**: Keep sensitive application images internal and avoid uploading them to the cloud.
- **Custom Integrations**: Integrate your CI/CD pipelines directly into your registry and enforce custom policies.
- **On-Prem Use Cases**: Ideal for air-gapped environments or regulatory compliance where public internet access is restricted.

### ⚠️ Considerations

- **Maintenance Overhead**: You are responsible for setup, updates, monitoring, and uptime.
- **Storage Management**: You need to plan for and maintain storage (e.g., via NFS, object storage, or persistent volumes).
- **TLS/Authentication Setup**: Setting up secure access via HTTPS and managing credentials is necessary.

> 🛠️ **A good self-hosted registry setup balances flexibility and security, while introducing operational responsibilities.**

---

## Popular Self-Hosted Registry Solutions

There are several well-supported options for running your own registry:

### Docker Registry (Official)
- ⚙️ The reference implementation maintained by the Docker team
- 🪶 Lightweight and easy to set up
- 🔐 Requires external authentication and UI layers if needed
- 📘 [Official Docs](https://distribution.github.io/distribution/)
- 📌 [My detailed blog setup for Docker Registry](https://www.ruchan.dev/Docker/PrivateRegistry/)

### Harbor
- 🌐 Full-featured, CNCF-hosted project by VMware
- 🔐 Built-in authentication, role-based access control (RBAC), vulnerability scanning, and image signing
- 🎛️ Web-based UI, Helm charts, and Kubernetes-friendly
- 📘 [Official Docs](https://goharbor.io/docs/)

### GitLab Container Registry
- 🧰 Integrated into GitLab CI/CD
- 💼 Perfect for GitLab-based teams
- 🔐 Uses GitLab users for authentication
- 📘 [GitLab Registry Docs](https://docs.gitlab.com/ee/user/packages/container_registry/)

### JFrog Artifactory
- 📦 Universal artifact manager that supports Docker, Helm, npm, etc.
- 🧩 Fine-grained access control and metadata
- ☁️ Can be run on-prem or via SaaS
- 📘 [JFrog Docs](https://jfrog.com/artifactory/)

---

## Final Thoughts

Container image registries play a critical role in modern DevOps and cloud-native workflows. Whether you’re using a public service like Docker Hub or running your own private registry, understanding how images are stored, tagged, and distributed empowers you to design scalable and secure software delivery pipelines.

📘 For hands-on setup instructions and configuration tips on running your own Docker Registry, check out my blog post: [Docker Private Registry Setup](https://www.ruchan.dev/Docker/PrivateRegistry/)

