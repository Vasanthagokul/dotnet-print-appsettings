## Project Setup

### Create the .NET Web Application:

```bash
dotnet new web -n dotnet-print-appsettings
cd dotnet-print-appsettings
```

### Update Program.cs:
```bash
code Program.cs
```
Replace the content with:
```csharp
using System;
using System.IO;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;

namespace dotnet_print_appsettings
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder
                        .UseStartup<Startup>()
                        .ConfigureAppConfiguration((hostingContext, config) =>
                        {
                            config.SetBasePath(Directory.GetCurrentDirectory());
                            config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
                            config.AddEnvironmentVariables(); // Add this line to load environment variables
                        });
                });
    }
}
```

### Create Startup.cs:
```bash
code Startup.cs
```
Add the following content:
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.IO;

namespace dotnet_print_appsettings
{
    public class Startup
    {
        public IConfiguration Configuration { get; }

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync($"Current Time: {System.DateTime.Now}\n");
                    await context.Response.WriteAsync($"Current Environment: {Configuration["Environment"]}\n");
                    await context.Response.WriteAsync($"Database Connection String: {Configuration["Database:ConnectionString"]}\n");
                });
            });
        }
    }
}
```

### Update appsettings.json:
```bash
code appsettings.json
```
Replace the content with:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Environment": "Development",
  "Database": {
    "ConnectionString": "local connection string"
  }  
}
```

### Create Dockerfile:
```bash
code Dockerfile
```
Add the following content:
```dockerfile
# Use the .NET SDK image as a build stage.
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build

# Set the working directory in the container.
WORKDIR /app

# Copy the project file and restore dependencies.
COPY *.csproj ./
RUN dotnet restore

# Copy the remaining files and build the application.
COPY . ./
RUN dotnet publish -c Release -o out

# Use the ASP.NET Core runtime image as the final stage.
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime

# Set the working directory in the container.
WORKDIR /app

# Copy the published app from the build stage.
COPY --from=build /app/out .

# Expose the port that the app is listening on.
EXPOSE 80

# Define the command to run the app when the container starts.
ENTRYPOINT ["dotnet", "dotnet-print-appsettings.dll"]
```

### Create .gitignore:
```bash
code .gitignore
```
Add the following content:
```
bin/
obj/
```

---

## Prerequisites for Ubuntu:

**Note: Use an Ubuntu VM for the following setup.**

### Install Docker:
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

### Install kubectl:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Install minikube:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### Install Helm:
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```

## Lesson 1: Basic Docker Deployment

### Build the docker image:
```bash
docker build -t dotnet-print-appsettings .
```

### Run the container:
```bash
docker run -d -p 8080:80 --name dotnet-print-appsettings-container dotnet-print-appsettings
```

Visit localhost:8080 and you will see a page where three things are printed:
- Current time
- Environment: this is fetched from appsettings.json
- Connection string: this is fetched from appsettings.json

![Output Screenshot](images/output-ss.png)

## Lesson 2: Docker Environment Variable Overrides

### Run with environment variable overrides:
```bash
docker run -d -p 8080:80 \
  -e Environment="Docker" \
  -e Database__ConnectionString="connection string from docker override" \
  --name dotnet-print-appsettings-container \
  dotnet-print-appsettings
```

![Docker Override Output Screenshot](images/docker-override-output.png)

## Lesson 3: Kubernetes Cluster Deployment

### Start minikube:
```bash
minikube start
```

### Install with Helm:
```bash
helm upgrade --install dotnet-print-appsettings ./helm-chart
```

### Port forward to access the service:
```bash
kubectl port-forward service/dotnet-print-appsettings-helm-chart 8080:80
```

**Note:** If the port-forward connection drops during helm operations or pod restarts, rerun the port-forward command above.

### The values in the appsettings.json are overriden by the values present in the helm chart
![K8s Override Output Screenshot](images/k8s-override-output.png)

## Using Different Values Files:

You can override values using different values.yaml files for different environments:

```bash
helm upgrade --install dotnet-print-appsettings ./helm-chart -f ./helm-chart/values-prod.yaml
```

```bash
helm upgrade --install dotnet-print-appsettings ./helm-chart -f ./helm-chart/values-staging.yaml
```

**Note:** If the application doesn't reflect the new values immediately, restart the deployment:
```bash
kubectl rollout restart deployment/dotnet-print-appsettings-helm-chart
```



## Centralized Helm Charts:

In production environments, it's common to use centralized Helm chart repositories. This allows teams to:
- Share common chart templates across multiple applications
- Maintain consistent deployment patterns
- Version control chart changes independently from application code
- Use tools like Helm repositories, ChartMuseum, or OCI registries for chart distribution

---

## Learning Outcomes

**Congratulations!** You have successfully completed this hands-on tutorial. Let's review what you've accomplished:

### ✅ What You've Learned:

**1. Containerization and Configuration Management**
- Built and deployed a .NET application using Docker
- Understood how applications read configuration from appsettings.json
- Learned to override configuration using environment variables

**2. Docker Environment Variable Injection**
- Used Docker's `-e` flag to override application settings
- Understood the `__` (double underscore) syntax for nested JSON properties
- Saw real-time configuration changes without rebuilding the application

**3. Kubernetes Deployments with Helm**
- Deployed applications to a Kubernetes cluster using Helm charts
- Managed different environments (staging, production) with values files
- Understood service discovery and port forwarding in Kubernetes

**4. DevOps Deployment Patterns**
- Experienced the progression from local development to containerized deployment
- Learned environment-specific configuration management
- Understood how to troubleshoot deployment issues (pod restarts, port-forward reconnections)

**5. Codebase Analysis Skills**
- Examined application structure (Program.cs, Startup.cs, appsettings.json)
- Understood how configuration flows through a .NET application
- Identified key files that DevOps engineers need to understand for deployment

### 🎯 Key Takeaways for DevOps Engineers:
- Configuration management is critical across different deployment environments
- Container orchestration requires understanding both the application and infrastructure
- Troubleshooting skills are essential when deployments don't behave as expected
- Version control and systematic approaches prevent deployment issues

### 🚀 Next Steps:
- Explore more complex Helm chart features (secrets, ingress, persistent volumes)
- Learn about CI/CD pipelines for automated deployments
- Practice with different application frameworks and their configuration patterns


