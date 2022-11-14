Steps to rebuild

1. Remove all Docker related items
- Dockerfile from all projects
- DOcker setting in launchSettings.json in all projects.
- dockerigrnore files in solution level (from file manager)

2. Disable SSL for simple run 
Startup.cs  // app.UseHttpsRedirection(); 
launchSettings.cs "sslPort": 0

3. Update correct local API url in Portal MVC app 
launchSetting.cs "applicationUrl": "http://localhost:54900",

Run the app and see the links point to API.

Once the app and API works locally, add Docker support manually.

4. Add Docker support to both apps (MVC and API)
Right click on the project > Add > Docker Support
Docker File Options Target OS: Linux or Window??
The target OS must match the current Docker Desktop OS setting. 
How to check? On the taskbar at th ebottom of the Windos screen, 
Find Docker Desktop Running icon. Right click on it.  If you see:
Switch to Windows Container --> currently Linux
Switch to Linux Container --> currently Windows

5. Build and test. Link doesnt work. MVC app needs to point to the new DOker API IP address.

6. Find the IP adress of Docker API. How?
> docker inspect containername
find ip address

Mamke sure the same envrionment (e.g. Development)

7. Replace the BaseUrl of CustomerServiceAPI with Docker IP.  

"CustomerServiceAPI": {
    //"BaseUrl": "http://main-api/",
    "BaseUrl": "http://172.17.0.2/",
    "APIUserName": "xxxx",
    "APIPassword": "xxxx"
  }
  --> it works, but when ip changes it wil breaks.

8. Solution is to use the Container name!
--> Doesnt work !!! 

I do not know the solution


#############################################
# Creating Container using Docker CLI
#############################################

cd C:\Users\Hando\Documents\DEVRoot\.NET\Docker\Docker.DotNetCoreDemo

# Create image for webapi
docker build --tag dotnetcoredemo:webapi --file .\Services\API\DNCD.Service.API\Dockerfile .

# create image for mvc
docker build --tag dotnetcoredemo:mvc --file .\Project\DNCD.Project.Portal\Dockerfile .

docker images

# create custom network
docker network ls
docker network create --driver bridge demo-mynetwork
docker network ls

# create a container for webapi image
docker run --name main-api --publish 3333:80 --network demo-mynetwork dotnetcoredemo:webapi

# create a container for mvc image
docker run --name main-mvc --publish 3336:80 --network demo-mynetwork dotnetcoredemo:mvc

Check the images and containers in Docker desktop. 

It works. 
###################################################
#  Put all these in Docker Compose (yaml file)
################################
See YAML file here:
C:\Users\Hando\Documents\DEVRoot\.NET\Docker\Docker.DotNetCoreDemo\docker-compose.yml

In Command
CD C:\Users\Hando\Documents\DEVRoot\.NET\Docker\Docker.DotNetCoreDemo
DIR to sse yml file exists

Run YAML file
> docker-compose UP --build
(Make sure to remove the previous created containers before run, 
Or, rename the containers and name and port number.
í.e. main-api-staging, main-mvc-staging
)

Add api url to portal appsettings.staging.json
"CustomerServiceAPI": {
    "BaseUrl": "http://main-api-staging/",

Che the containers in Docker Desktop. and Test. 

####################################################################################
####  CI/CD pipeline for DOcker Container | Azure DevOps for Docker Containers
####################################################################################

### Create Azure Container Registry
> Azure Portal > Container registries > Create

Registry Name: liamlim  (--> liamlim.azurecr.io)
SKU: Basic (It costs!!)

# Step 1. Create a docker images in local

CMD>
CD C:\Users\Hando\Documents\DEVRoot\.NET\Docker\Docker.DotNetCoreDemo

# Create webapi image
docker build --tag dotnetcoredemo:webapi --file .\Services\API\DNCD.Service.API\Dockerfile .

# Create mvc image
docker build --tag dotnetcoredemo:mvc --file .\Project\DNCD.Project.Portal\Dockerfile .

# Step 2. Tag local docker image
docker tag dotnetcoredemo:webapi liamlim.azurecr.io/dotnetcoredemo-webapi:v1
docker tag dotnetcoredemo:mvc liamlim.azurecr.io/dotnetcoredemo-mvc:v1
 
# note. dotnetcoredemo-webapi  --> repository name
        dotnetcoredemo-mvc  --> repository name
        v1  --> tag 

# step 3. login to Azure container registry
docker login liamlim.azurecr.io --username liamlim --password RZYc5KRC=fTbVtgfELrjNRYLroHO=ya7

# step 4. push lock docker images to Azure container registry
docker push liamlim.azurecr.io/dotnetcoredemo-webapi:v1
docker push liamlim.azurecr.io/dotnetcoredemo-mvc:v1

# step 5. Creats container instances

### Create Azure Container instances
> Azure Portal > Container instances > Create

# API Container Instance

Container name: demo-api
Image source: Azure container registry
Registry: liamlim
Image: dotnetcoredemo-webapi
Image tag: v1
Networking
    networking type: public
    DNS name label: demo-staging-api
    port: default 80
Advanced
    Environmental Variables
    Key: ASPNETCORE_ENVIRONMENT
    Value: Staging
    --> this allows the process to access addpsetting.Staging.json

Once the instance is created, test to browse the api. (Add api/customers to the url) 
demo-staging-api.australiaeast.azurecontainer.io/api/customers

# MVC Container Instance

Update appsetting.staging.json
"BaseUrl": "http://demo-staging-api.australiaeast.azurecontainer.io/" 
,or
"BaseUrl": "http://demo-staging-api.abc6agf6cdbkhsab.australiaeast.azurecontainer.io/",

Create new image with above update and push again.
docker build --tag dotnetcoredemo:mvc --file .\Project\DNCD.Project.Portal\Dockerfile .
docker push liamlim.azurecr.io/dotnetcoredemo-mvc:v2
NOTE. tag v2

Container name: demo-mvc
Image source: Azure container registry
Registry: liamlim
Image: dotnetcoredemo-mvc
Image tag: v2  (NOTE!!!)
Networking
    networking type: public
    DNS name label: demo-staging-mvc
    port: default 80
Advanced
    Environmental Variables
    Key: ASPNETCORE_ENVIRONMENT
    Value: Staging
    --> this allows the process to access addpsetting.Staging.json

Once the instance is created, test to browse the api. 
demo-staging-mvc.australiaeast.azurecontainer.io/






