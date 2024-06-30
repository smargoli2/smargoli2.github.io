---
layout: post
title: "GUIDE: Chargeback Feature"
date: 2024-06-27 10:33:15 +0300
categories: Guide Oodle
---

## Chargeback Feature Guide

A chargeback is a transaction for a credit that a vendor has to give to Chefman for various reasons like a cancelled PO or merchandise issue. Think of it as an invoice with line items representing each transaction.

# Create the database tables

Write a sql script to create 2 tables in the database. Save this script in the 'next' folder, since it will be used during deployment to update the demo and production databases.
[https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders/?path=%2Fnext&version=GBtrunk&_a=contents](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders/?path=%2Fnext&version=GBtrunk&_a=contents)

{% highlight sql %}
Chargeback: Id, CreatedOn, CreatedBy, LastModified, LastModifiedById, VendorId, Notes, SubsidiaryId
ChargebackLine: Id, ChargebackId, Notes, Amount, PoId, PoNumber
{% endhighlight %}

# Create classes for the entities
Create a classes for the `Chargeback` and `ChargebackLine` entities in the OodleEntities project: [https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/OodleEntities&version=GBtrunk&_a=contents](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/OodleEntities&version=GBtrunk&_a=contents)

The Chargeback class should implement `Entity` and `IDatedChanges<string>` which includes the columns like CreatedOn, CreatedBy, LastModifiedById and LastModified.
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

You will need to create a new permission type called `AddOrUpdateChargeback`. Add a line to the sql script to insert a new row for the permission into the Permissions table.

Add the new permission to the [PermissionNames.cs](https://oodle.visualstudio.com/Web%20App/_git/PurchaseOrders?path=/src/BaseModels/PermissionNames.cs&version=GBtrunk&_a=contents) class.

Now you can use the permission on the controller endpoints to restrict who can call the endpoints.

The create/update endpoints take a DTO from the front end and use their base class, the `BaseEntityController` to take care of creating/updating the items in the database.

# Testing

You can test this endpoint using [Postman](https://www.postman.com/). You can use it online or in the desktop app.
Run Oodle locally.
In order to authenticate yourself with Oodle, you need to first make a Postman request to get an XSRF token. You then pass this token along with your other requests, and this tells Oodle that you are authenticated.
Here's a link to the request: [Get XSrf token](https://cloudy-capsule-944641.postman.co/workspace/New-Team-Workspace~5cc1fcc0-162b-4f80-96ad-89a5c2aa7976/request/23443504-37025919-32aa-4578-8fc6-11dae92162f6?action=share&source=copy-link&creator=23443504&ctx=documentation)

Send the reuest. The Cookies tab of the response will have a cookie called XSRF-TOKEN. Copy the value, and you will use it to call other endpoints in Oodle.
![alt text](image.png)
Take a look at the the Put SO on Hold request. 
![alt text](image-1.png)
The XSRF token is passed as a header in the request. Create a new request to call the CreateChargeback endpoint, with the POST method and url of http://localhost:5000/api/Chargeback/Create.
The Content-Type is `application/json`. In the Body tab, select 'raw' and choose json from the dropdown.
Enter the JSON representation of the SteamshipLineDTO. For example:
{% highlight json %}
{
  "property1": "value1",
  "property2": "value2",
  ...
}
{% endhighlight %}