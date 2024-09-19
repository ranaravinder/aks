
---

# Sending Email using Managed Identity and Microsoft Graph API in .NET

This guide explains how to use Azure **Managed Identity** with Microsoft Graph API to send emails in a .NET application, without the need to manage client secrets.

## Prerequisites

1. **.NET SDK** installed on your machine.
2. Azure resource (like App Service, Function App) with **Managed Identity** enabled.
3. Azure Active Directory permissions to use **Microsoft Graph API** for sending emails.

## Step 1: Enable Managed Identity in Azure

1. Go to your **Azure resource** (e.g., App Service, Function App).
2. Under the **Identity** section, enable **System-assigned Managed Identity**.

![Enable Managed Identity](https://docs.microsoft.com/en-us/azure/app-service/media/app-service-managed-service-identity/app-service-managed-service-identity-enable.png)

## Step 2: Assign API Permissions to Managed Identity

1. Go to **Azure Active Directory** → **Enterprise Applications** → find the Managed Identity for your resource.
2. In the **API Permissions** section, assign the **Mail.Send** permission for **Microsoft Graph**.
   - Ensure the permission is granted by an admin.

## Step 3: Install Required NuGet Packages

Make sure you have the following NuGet packages installed in your .NET project:

```bash
dotnet add package Azure.Identity
dotnet add package Microsoft.Graph
```

## Step 4: Code to Send Email Using Managed Identity

Below is the code to send an email using **Managed Identity** in a .NET application.

```csharp
using Azure.Identity;
using Microsoft.Graph;
using Microsoft.Graph.Models;
using Microsoft.Graph.Users.Item.SendMail;
using System.Collections.Generic;
using System.Threading.Tasks;

public static class GraphEmailSender
{
    public static async Task SendEmailUsingManagedIdentityAsync()
    {
        // Create Managed Identity Credential
        var managedIdentityCredential = new ManagedIdentityCredential();
        var graphClient = new GraphServiceClient(managedIdentityCredential);

        // Create email message
        var body = new SendMailPostRequestBody
        {
            Message = new Microsoft.Graph.Models.Message
            {
                Subject = "This is the subject line",
                Body = new ItemBody
                {
                    ContentType = BodyType.Text,
                    Content = "This is the body of the email message."
                },
                ToRecipients = new List<Recipient>
                {
                    new Recipient
                    {
                        EmailAddress = new EmailAddress
                        {
                            Address = "recipient-email@example.com" // Recipient's email address
                        }
                    }
                }
            }
        };

        // Send the email using the Graph API
        await graphClient.Users["your-email@example.com"].SendMail.PostAsync(body);

        // Optionally handle the response here
    }
}
```

### Key Changes

- **ManagedIdentityCredential** replaces **ClientSecretCredential** for authentication.
- No need to supply tenant ID, client ID, or client secret, as Managed Identity handles this automatically in Azure.
  
## Step 5: Test Locally with Azure CLI

To test this locally, you can use Azure CLI to simulate Managed Identity:

```bash
az login
```

Once logged in, the **ManagedIdentityCredential** will authenticate using your Azure CLI session when running locally.

---

## Conclusion

This approach eliminates the need to manage credentials manually in your application. By leveraging Azure **Managed Identity**, your application can securely authenticate with Microsoft Graph API and send emails without embedding sensitive information like client secrets.

