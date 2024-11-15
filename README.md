# umbraco-blog

Check dotnet version installed, needs to be dotnet 8 sdk or higher for latest version of umbraco
- dotnet --list-sdks
 
Install Visual Studio community

Install version 8 using winget
- winget install dotnet-sdk-8

Install node 20.11.0 or higher, lts is 22.11.0
- install node version manager (nvm)
- https://github.com/coreybutler/nvm-windows#installation--upgrades
- nvm list available
- nvm install lts
- nvm use 22.11.0, if exit status 5: access is denied lauch an admin shell and then run the command

Database setup
- db_owner role
- or db_datareader and db_datawriter and db_ddladmin for modifying the db schema
- might need to show how to install it
- install sql server developer edition 2022 https://www.microsoft.com/en-us/sql-server/sql-server-downloads
- install sql server management studio https://learn.microsoft.com/en-gb/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver16#download-ssms
- open sql server management studio
- create database called umbraco
- goto server > security logins > new login
- login name umbraco-admin, SQL server auth
- user mapping select umbraco database
- select role membership as db_owner, leave public ticked, click ok

Install Umbraco templates
- dotnet new install Umbraco.Templates::13.*
- dotnet new umbraco --name website
- cd website
- dotnet run
- https://localhost:44314 connection not private, error
- dotnet dev-certs https --trust
- install wizard
- select SQL server
- server address localhost
- enter db credentials for db setup
- use integrated auth and trust certificate
- click install
- Welcome to umbraco

Install articulate
- https://marketplace.umbraco.com/
- Themes & Starter Kits
- Select Articulate > https://marketplace.umbraco.com/package/articulate
- https://github.com/Shazwazza/Articulate/wiki
- Stop umbraco, ctrl + c
- then run dotnet add package Articulate... doesn't support umbraco 14 :(
- dotnet remove package Articulate
- dotnet run
- localhost:44373/umbraco
- will now see a blog under content, we need to publish this before we can see it
- issue with the blog root banner and logo, require the document type to be resaved can then publish
  - settings > Document Types > Articulate > blogBanner + blogLogo
- 4 installed themes 
  - go to blog under content scroll down to style section and select theme you want from theme dropdown list; material, mini, vapor, phantom
- create a custom theme by copying an existing theme folder under go to back office > Articulate > Articulate Themes > Select theme to copy and give it a new name
  - theme files are copied under Views > ArticulateThemes > Theme Name
	- You can now edit this and customise it to your needs
- commit changes and files added by umbraco and articulate to GitHub.
- umbraco ships with a default .gitignore
- in the terminal we can do git add . in the root dir and then git commit -m "initial commit" and git push
- or in the right menu select Git Changes and commit all with a message, then push the up arrow icon

Host on Azure
- you will need an azure account
  - free trial available which gives you USD200 for 30 days, needs a credit card or you can go with payg
	- https://azure.microsoft.com/en-gb/pricing/purchase-options/azure-account?icid=azurefreeaccount&ref=VisualStudio
- Visual Studio Professional �1378.00 / �918 renewal https://www.microsoft.com/en-gb/d/visual-studio-professional-subscription/dg7gmgf0dst3/0001?activetab=pivot:overviewtab
  - Includes Visual Studio Professional
  - $50 of Azure credits each month
  - 3 months of pluralsight
  - dev copies of microsoft software like windows server

Go to App Services
- https://portal.azure.com/#browse/Microsoft.Web%2Fsites
- Create Web App + Database
- Project
  - resource group: rg-umbraco
  - region: UK South or region closest to you
  - website name: cn-umbraco, turn off unique default hostname
  - runtime: .NET 8
- Database
  - SQLAzure
  - servername: use default based on web ap name cn-umbraco-server
  - databasename: cn-umbraco-database
  - add Azure Cache for Redis: no
- Hosting: Basic
- Review + Create > create
- 
db username: cn-umbraco-server-admin
password: xxxxx

- deployment will take a few minutes
- once complete go to resource
- click on the browse link, might take a little while to start up... not showing anything...
- look at resource group for what else was deployed, we get a vnet, private link and server/database
- 
Deployment center
- In deployment center go to settings and select source, pick GitHub
- signin with GitHub account, may autodetect
- select org: mike-w-kelly (username)
- Repo: umbraco-blog
- branch: main
- auth: user-assign identity
- sub: your azure sub
- identity: create new
- preview file which will create a github actions workflow and save in the .github folder in your repo
- click save
- refresh logs to show first deployment
- will fail, click on Build/Deploy Logs link to take you to the build in GitHub actions
- on the left click on the workflow file under run details
- click on the pencil icon in the right corner to edit the workflow file
- for the Build with dotnet & dotnet publish add a working-directory: ./website which is where our .csproj file lives
- commit the changes which will retrigger the build and deploy the webapp, this will take a few mins

Prepare to export local database to Azure SQL set firewall
- Go to Azure Portal
- go to sql databases
- select our database
- Set server firewall
- add your client IPv4 addess to the firewall rules, seems to be a bug which shows the IPv6 address so google what's my ip and use your address
- click save
- Open SQL Server Management Studio
- connect to Database engine > server name server-name.database.windows.net, the SQL Server authentication use username and password from when the database was created

Import / Export Data tool
- Open Visual Studio Installer, click modify installation
- Click individual components
- serach for sql server data tools 
- if not already installed click Data sources for SQL Server support and SQL Server Data Tools. Then install
- If already present exit

Export data
Source
- right click on database
- Tasks > Export data...
- click next
- Data source: SQL Server Native Client
- Server name: (local)
- Authentication: Windows
- Database: umbraco
Destination
- SQL Server Native Client
- Servername: servername.database.windows.net
- Use SQL Server Auth
- username and password of the sql server
- Database name: cn-umbraco-database or your name
- click next
- select copy data from one or more tables or views, click next
- select all tables and click next
- click next
- keep run immediately selected and click next
- review the actions and click finish
- Will perform the operation, once the operation has successfully completed click close

Grant access to App Service using Managed Identity
- on the app service click on the resource group link
- under the resource group you will see the managed identity that was created for the app service when we did the original deployment
- click the managed identity
- select Azure role assignments
- you can see that it currently has Website Contributor
- Next click + Add role assignment (Preview)
- for scope click SQL, then select your subscription
- for resource click on the sql server we've created earlier
- for Role select Owner to keep this simple
- Then save, after a minute or two you'll see the Owner role in the list for the managed identity below the Website Contributor

Update the connection string
- In Visual Studio
- Make a copy of the appsettings.json and call it appsettings.Release.json
- add in the app service specific settings as well
- delete everything except the connection string
- Then Azure go to the database
- Under settings > connection strings copy the first entry for ADO.NET (Microsoft Entra passwordless authentication)
- You may need to copy/type this manually as Visual Studio does something weird when it pastes it. You'll need to escape the quotes around the Active Directory Default text with a backslash \"
- commit and push the changes
- check github actions for deployment status

Check site is running
- go to app service and click browse
- if already open refresh the page
- If everything worked we should see the working site
- can see progress of container starting in app service > monitoring > Log stream

