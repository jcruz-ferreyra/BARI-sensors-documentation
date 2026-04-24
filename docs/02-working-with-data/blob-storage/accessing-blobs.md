# Accessing Blob Storage

This page covers how to connect to and interact with Azure Blob Storage for inspecting, downloading, uploading, or moving sensor data blobs. For the storage organization and folder structure, see [Blob Storage Reference](../../05-reference/azure-infrastructure/blob-storage.md).

## Prerequisites

**1. Azure Account Access**

You need an Azure account with appropriate permissions to access the storage account resource. Unlike the database, blob storage access is typically reserved for technical administrators performing maintenance or data corrections — not for general research queries.

**2. Authentication Method**

One of the following:

- **Microsoft Entra ID (formerly Azure AD)** - Uses your Azure organizational account (used with Azure Storage Explorer)
- **Managed Identity** - Automatic authentication for Azure resources (used by Function Apps)
- **Connection String** - Direct access key (used for local Python scripts)

See [Authentication Methods](#authentication-methods) below for detailed explanations.

**3. Network Access**

The storage account uses private networking by default. You need either:

- Access from within the Azure VNet (automatic for Azure resources)
- Temporary public access enabled from the Azure Portal (see [Network Access](#network-access) below)

For more details on the networking architecture, see [Networking Reference](../../05-reference/azure-infrastructure/networking.md).

!!! info "Need Help?"
    If you're unable to meet any of these prerequisites (Azure access, authentication setup, or network configuration), contact {{ contacts.technical_administrator.name }} ({{ contacts.technical_administrator.role }}).

---

## Connection Methods

Choose a connection method based on your task. Azure Storage Explorer is best for visual inspection and one-off operations, while Python scripts are better for repetitive or bulk tasks.

### Azure Storage Explorer (Recommended for Visual Interaction)

**What it is:** Desktop application for browsing and managing Azure Storage resources with a visual file-explorer interface.

**Best for:** Inspecting blob contents, browsing folder structure, quick manual downloads or uploads, verifying pipeline behavior.

**Who can use this:** Technical administrators with Azure account access. Requires sign-in with Northeastern credentials through the application.

**Download:** [Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer/)

**Limitation:** Not practical for bulk or scripted operations.

??? note "Step-by-Step: Installing and Connecting with Azure Storage Explorer"

    **Installation**

    Download and install from the link above. Available for Windows, Mac, and Linux.

    ---

    **Connecting to the Storage Account**

    **1. Sign In**

    When you first open Azure Storage Explorer, click "Connect to Azure resources" or use the plug icon in the left sidebar.

    ![Storage Explorer Connect Dialog](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_connect_dialog.jpg)

    Select "Subscription" as the resource type, then sign in with your Northeastern Azure credentials.

    ![Storage Explorer Sign In](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_sign_in.jpg)

    **2. Locate the Storage Account**

    After signing in, expand your subscription in the left panel. Navigate to: Storage Accounts → sensordatastprdue201 → Blob Containers → nu-sensor-data.

    ![Storage Explorer Navigation](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_navigation.jpg)

    **3. Browse Folders**

    The folder structure mirrors the [Blob Storage Reference](../../05-reference/azure-infrastructure/blob-storage.md#folder-structure). Click into any folder to see its contents.

    ![Storage Explorer Folder Structure](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_folder_structure.jpg)

    ---

    **Common Operations**

    **Viewing Blob Contents**

    Double-click any blob to download and open it. JSON files open in your default text editor.

    ![Storage Explorer Blob Preview](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_blob_preview.jpg)

    **Downloading Blobs**

    Select one or more blobs → Click "Download" in the toolbar → Choose destination folder.

    ![Storage Explorer Download](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_download.jpg)

    **Uploading Blobs**

    Navigate to the target folder → Click "Upload" → Select files from your computer.

    ![Storage Explorer Upload](../../images/screenshots/02-working-with-data/blob-storage/accessing/explorer_upload.jpg)

    **Moving Blobs**

    Azure Storage Explorer does not support moving blobs directly. To move a blob, download it, upload it to the new location, then delete the original. For bulk moves, use the [Python scripts](#python-scripts-recommended-for-bulk-operations) instead.

    ---

    See [Network Access](#network-access) for enabling public access before connecting.

<div style="margin-top: 2rem;"></div>

### Python Scripts (Recommended for Bulk Operations)

**What it is:** Python scripts using the `azure-storage-blob` SDK to interact with blob storage programmatically. The project maintains a repository of common maintenance scripts for this purpose.

**Best for:** Bulk uploads, batch downloads, moving blobs between folders, any repetitive or scriptable task.

**Who can use this:** Anyone comfortable with Python programming who has the storage account connection string.

**Repository:** [Manual Tasks Repository]({{ urls.manual_tasks_repo }}) (private — requires organization GitHub access)

**Installation:** `pip install azure-storage-blob python-dotenv`

**Example tasks the scripts handle:**

- Uploading blobs from text files collected manually from sensor SD cards into the `incoming/` folder for database processing
- Downloading blobs matching a pattern (for example, all blobs for a specific box_id from a specific folder)
- Moving blobs from one folder to another (for example, moving blobs from `failed-processing/` back to `incoming/` after investigating an issue)

??? note "Step-by-Step: Setting Up Python Blob Access"

    **1. Clone the Manual Tasks Repository**

    Clone the repository from the organization's GitHub account. The repository contains ready-to-use scripts for common blob operations.

    **2. Create a `.env` File**

    In the repository root, create a file named `.env` and add the storage account connection string:

    ```
    AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...;EndpointSuffix=core.windows.net
    ```

    The connection string can be found in Azure Portal → Storage Account → Access keys.

    ![Azure Portal Access Keys](../../images/screenshots/02-working-with-data/blob-storage/accessing/portal_access_keys.jpg)

    **3. Verify `.env` Is in `.gitignore`**

    The `.env` file contains the full connection string which grants direct access to the storage account. Confirm that `.env` is listed in the repository's `.gitignore` file so it is never committed to version control.

    **4. Enable Public Access**

    Before running scripts from your local machine, enable temporary public access on the storage account and add your IP to the firewall. See [Network Access](#network-access) below.

    **5. Run a Script**

    Each script in the repository uses `argparse` for command-line arguments. Run with `--help` to see available options.

!!! warning "Environment Variables and Secrets"
    The connection string provides full access to the storage account — anyone who has it can read, write, and delete blobs. This is why environment variables and `.env` files are used: they keep secrets out of your code and out of version control. For a quick primer on why this matters and how `.env` files work, see [python-dotenv documentation](https://pypi.org/project/python-dotenv/).

    **Never commit `.env` files or connection strings to Git.** If a connection string is accidentally exposed, rotate the storage account keys immediately in the Azure Portal.

---

## Authentication Methods

The storage account supports three authentication methods. Choose based on your use case.

### Microsoft Entra ID (Used with Azure Storage Explorer)

Uses your Northeastern Azure credentials to authenticate through Azure Storage Explorer. Access depends on the role assignments configured for your account on the storage account resource.

**Benefits:**

- No keys or passwords to manage locally
- Permissions tied to your organizational account
- Access automatically revoked when leaving organization

**Limitation:** Currently, the storage account is configured for technical administrator access. Reader-level role assignments for other team members have not been set up, since blob storage interaction is primarily a maintenance activity.

**How to connect:** Follow the [Azure Storage Explorer steps](#azure-storage-explorer-recommended-for-visual-interaction) above.

### Connection String (Used with Python Scripts)

A connection string containing the storage account access key. Provides full read/write access to all containers and blobs.

**Benefits:**

- Simple setup for local scripts
- Works with the `azure-storage-blob` Python SDK
- No Azure CLI or browser sign-in required

**Risks:**

- The connection string grants full access — treat it like a password
- Must be stored in `.env` files, never in code or version control
- If exposed, rotate storage account keys immediately

**How to connect:** Follow the [Python scripts setup steps](#python-scripts-recommended-for-bulk-operations) above.

### Managed Identity (For Azure Resources Only)

Azure resources within the same VNet (Function Apps) authenticate automatically without storing credentials. This is how the Webhook Receiver and Database Writer access blob storage in production.

**How it works:** The Function App's managed identity is granted the appropriate role on the storage account. Combined with private endpoint access through the VNet, the function connects without any keys or connection strings in code.

**Configuration:** See [Function Apps - Authentication](../../05-reference/azure-infrastructure/function-apps.md#authentication) for setup details.

!!! warning "Network Access Required"
    Both Microsoft Entra ID and Connection String authentication require network connectivity to the storage account. If public network access is disabled (default), connections from outside Azure will fail regardless of authentication method.

    See [Network Access](#network-access) below and [Networking Reference](../../05-reference/azure-infrastructure/networking.md) for details on enabling temporary public access.

    **Exception:** Managed Identity works through private endpoints and does not require public access.

---

## Network Access

The storage account uses private networking by default, restricting connections to Azure resources within the same Virtual Network (VNet). The same concept applies here as with the [database](connecting.md#network-access) — public access must be temporarily enabled when working from a local machine.

### Private Endpoint (Default)

**What it means:** The storage account is only accessible from within Azure's internal network.

**Who can connect:**

- Azure Function Apps (automatically — they're in the VNet)
- Other Azure resources configured in the same VNet

**Who cannot connect:**

- Your laptop/desktop computer
- Azure Storage Explorer running locally
- Python scripts running on your local machine

For complete networking architecture details, see [Networking Reference](../../05-reference/azure-infrastructure/networking.md).

### Temporary Public Access

To connect from your computer (whether through Azure Storage Explorer or Python scripts), you need to temporarily enable public network access.

**Security implications:**

- Opens storage account to internet connections (with firewall restrictions)
- Should only be enabled when actively working with blobs
- Must be disabled immediately after use
- Always restrict access to your specific IP address only

### Enabling Temporary Public Access

??? note "Step-by-Step: Enable Public Access for Storage Account"
    1. Navigate to Azure Portal → Storage Account (sensordatastprdue201)
    2. Select "Networking" from the left sidebar under "Security + networking"
    3. Under "Public network access" → Select "Enabled from selected virtual networks and IP addresses"
    4. Under "Firewall" → Click "Add your client IP address"
    5. Click "Save"

    ![Storage Account Networking Settings](../../images/screenshots/02-working-with-data/blob-storage/accessing/portal_networking_settings.jpg)

    **To disable:** Return to same page, select "Disabled" under "Public network access", click "Save"

For detailed instructions with screenshots, see [Appendix: Network Configuration](../../06-appendices/step-by-step-guides/configure-network-access.md).

!!! warning "Remember to Disable"
    Always disable public access when finished. Leaving it enabled creates unnecessary security exposure — especially for blob storage, where the connection string grants full read/write access.


<!-- Here's a summary of the 9 screenshot placeholders you'll need to capture, all under images/screenshots/02-working-with-data/blob-storage/accessing/:

explorer_connect_dialog.jpg — The "Connect to Azure resources" dialog
explorer_sign_in.jpg — Subscription selection and Entra sign-in flow
explorer_navigation.jpg — Left panel expanded to nu-sensor-data container
explorer_folder_structure.jpg — Folder listing inside the container
explorer_blob_preview.jpg — A blob opened/previewed showing JSON contents
explorer_download.jpg — Download toolbar action with blob(s) selected
explorer_upload.jpg — Upload dialog
portal_access_keys.jpg — Azure Portal Storage Account → Access keys page
portal_networking_settings.jpg — Azure Portal Storage Account → Networking page with firewall settings -->