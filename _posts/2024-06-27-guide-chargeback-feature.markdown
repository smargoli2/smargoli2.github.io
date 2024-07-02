---
layout: post
title: "GUIDE: Chargeback Feature"
date: 2024-06-27 10:33:15 +0300
categories: Guide Oodle
---

## Chargeback Feature Guide

A chargeback is a transaction for a credit that a vendor has to give to Chefman for various reasons like a cancelled PO or merchandise issue. Think of it as an invoice with line items representing each transaction. A chargeback is created for a vendor and then approved by a manager.

# Create the database tables

Write a sql script to create the new tables in the database. Save this script in the 'next' folder, since it will be used during deployment to update the demo and production databases.
[https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders/?path=%2Fnext&version=GBtrunk&_a=contents](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders/?path=%2Fnext&version=GBtrunk&_a=contents)

{% highlight sql %}
Chargeback: Id, CreatedOn, CreatedBy, LastModified, LastModifiedById, VendorId, Notes, SubsidiaryId, CurrentStateValue
ChargebackLine: Id, ChargebackId, Notes, Amount, PoId, PoNumber
{% endhighlight %}

Create a table to hold the status changes. They way we keep track of entity approvals is by adding a row to a table each time the entity status is updated.

See the PurchaseOrder_Status table for an example and copy the columns from there.

# Create classes for the entities
Create a classes for the `Chargeback` and `ChargebackLine` entities in the OodleEntities project: [https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/OodleEntities&version=GBtrunk&_a=contents](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/OodleEntities&version=GBtrunk&_a=contents)

The Chargeback class should implement `Entity` and `IDatedChanges<string>` which includes the columns like CreatedOn, CreatedBy, LastModifiedById and LastModified. It should also implement `IApprovable<Chargeback, int, string>`, which includes the CurrentStatusValue property and helps the entity implement the approval process.
The ChargebackLine should implement `Entity` and `IParentDatedChanges<string>` (the change history is on the parent).

Add all the fields that we added to the database tables.

We also need to create a DTO class in the Models project, which stands for Data Transfer Object, which is used when transferring data, often from the front end to the back end and vice versa.

[https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/Models&version=GBtrunk&_a=contents](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/Models&version=GBtrunk&_a=contents)

Create a `ChargebackDTO` and `ChargebackLineDTO` and add all the fields. The `Chargeback` can implement `IHasIntId` and `IDtoHasCreatedBy`, and the `ChargebackLineDTO` can implement `IHasIntId`.

Now we need to tell C# how to map the entity to the DTO and vice versa. We use a library called [AutoMapper](https://docs.automapper.org/en/stable/Getting-started.html) which makes mapping objects very simple. Automapper can infer which columns match which if they have the same column name.

We have a class called `MapperProfile` that AutoMapper looks to for instructions. Add the following code [here](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/WebApp/MapperProfile.cs&version=GBtrunk&line=1725&lineEnd=1725&lineStartColumn=9&lineEndColumn=78&lineStyle=plain&_a=contents):

{% highlight C# %}
OodleCreateMap<Chargeback, ChargebackDTO>().ReverseMap();
OodleCreateMap<ChargebackLine, ChargebackLineDTO>().ReverseMap();
{% endhighlight %}

This tells AutoMapper that the classes can be mapped to eachother, and the types line up.

# MVC Controller Endpoint

MVC is a design pattern used to decouple user-interface (view), data (model), and application logic (controller). This pattern helps to achieve separation of concerns. [source](https://dotnet.microsoft.com/en-us/apps/aspnet/mvc#:~:text=Model%20View%20Controller%20(MVC),to%20achieve%20separation%20of%20concerns.)

You now need to create the `Controller` class with the endpoint for the front end to call to create the chargeback.
Use the `SteamshipLineController` as a base for your class. [link](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/WebApp/Controllers/SteamshipLineController.cs&version=GBtrunk&_a=contents)

Make endpoints for `GetAll()`, `Create()` and `Update()`.

You will need to create new permissions types called `GetChargebacks`, `AddOrUpdateChargeback` and `ApproveChangeback`. Add code to the sql script to insert new rows for the permissions into the Permissions table.

Add the new permissions to the [PermissionNames.cs](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/BaseModels/PermissionNames.cs&version=GBtrunk&_a=contents) class.

Now you can use the permissions on the controller endpoints to restrict who can call the endpoints.

The create/update endpoints take a DTO from the front end and then call the `Service` class to take care of creating/updating the items in the database.

Create a `ChargebackService` that implements `IBaseService<Chargeback>`, `IDatedChangesRepo<Chargeback, string>` and `IApprovableService<Chargeback>`. The BaseService provides lots of base functionality like creating and updating entities. The `DatedChangesRepo` (Repo is the old name we used to use instead of Service) returns entities modified/deleted by date, and the `ApprovableService` provides the `ApproveAsync()` method for approving entities.

Create an endpoint for approving a chargeback. Take a look at the `Approve()` endpoint for purchase orders: [PurchaseOrdersController](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/WebApp/Controllers/PurchaseOrdersController.cs&version=GBtrunk&line=135&lineEnd=135&lineStartColumn=9&lineEndColumn=57&lineStyle=plain&_a=contents). It calls the base approvable service's approve method to approve the entity.


Create an endpoint for GetChargebacksByVendorId. Check out how the [PurchaseOrderRepo](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/OodleDomain/Services/PurchaseOrderService.cs&version=GB2769_Queue_Hosted_Service_Graceful_Shutdown&line=238&lineEnd=238&lineStartColumn=51&lineEndColumn=80&lineStyle=plain&_a=contents) gets the purchase orders by model id: 
# Other Setup
The `InventoryDbContext` is an Entity Framework class that manages the database connection and operations, allowing you to query and save data. It represents a session with the database and provides methods to handle CRUD operations.

The status tables are set up using a special configuration which tells entity framework that there is a table called Entity_Status and it has certain columns. To register the Chargeback_Status table, add this to the [InventoryDbContext](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=%2Fsrc%2FOodleEntities%2FInventoryDbContext.cs&version=GBtrunk&_a=contents) around line 91:

{% highlight C# %}
modelBuilder.ApplyConfiguration(new EntityStatusConfiguration<Chargeback>());
{% endhighlight %}

The Startup class configures services and the app's request pipeline, defining how the app should respond to HTTP requests. It typically contains methods like ConfigureServices and Configure for setting up dependency injection and middleware. Google these terms and learn more about them.

Register the new `ChargebackService` in [Startup.cs](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/WebApp/Startup.cs&version=GBtrunk&_a=contents) around line 538.

`services.AddScoped<IBaseService<Chargeback>, ChargebackService>();`

# Testing

You can test the endpoints using [Postman](https://www.postman.com/). You can use it online or in the desktop app.
Run Oodle locally.

You will first need to grant yourself the new permissions that you created. Navigate to the settings in Oodle:

![alt text](/docs/assets/images/oodle-settings.png)

Go to the Roles Settings tab. You already have the Admin role, but you need to add the new permissions to the admin role. Click the 3-dot menu on the Admin user and click Permissions.
Check off all the permissions that are not checked or that have a dash in the checkbox. This gives you all permissions in your local development database so you can work unhindered. Restart the app so that the changes take effect (users and permissions are cached so you need to restart the app for the cache to be refreshed).

In order to authenticate yourself with Oodle and call an endpoint via Postman, you need to first make a Postman request to get an XSRF token. You then pass this token along with your other requests, and this tells Oodle that you are authenticated.
Here's a link to the request: [Get Xsrf token](https://cloudy-capsule-944641.postman.co/workspace/New-Team-Workspace~5cc1fcc0-162b-4f80-96ad-89a5c2aa7976/request/23443504-37025919-32aa-4578-8fc6-11dae92162f6?action=share&source=copy-link&creator=23443504&ctx=documentation)

Send the request. The Cookies tab of the response will have a cookie called XSRF-TOKEN. Copy the value, and you will use it to call other endpoints in Oodle.

![alt text](/docs/assets/images/cookie-xsrf.png)

Take a look at the the Put SO on Hold request. 

![alt text](/docs/assets/images/so-hold.png)
The XSRF token is passed as a header in the request. Create a new request to call the CreateChargeback endpoint, using the POST method and url of `http://localhost:5000/api/Chargeback/Create`.
The Content-Type is `application/json`. In the Body tab, select 'raw' and choose json from the dropdown.
Enter the JSON representation of the SteamshipLineDTO. For example:
{% highlight json %}
{
  "property1": "value1",
  "property2": "value2",
  ...
}
{% endhighlight %}
If your request is successful, you will see a status of 200 OK and the response body will have the newly created chargeback with an ID.