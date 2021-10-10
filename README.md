# Graph Onedrive

Interact with Microsoft OneDrive using the Graph API through an object.

## License

This project itself is subject to BSD 3-Clause License detailed in [LICENSE](https://github.com/dariobauer/graph-onedrive/blob/main/LICENSE).

The Graph API is provided by Microsoft Corporation and subject to their [terms of use](https://docs.microsoft.com/en-us/legal/microsoft-apis/terms-of-use).


## Installation

General installation is using pip from PyPI, but depending on your installation you may need to use `pip3` instead.

    pip install graph-onedrive

You can also install the in-development version:

    pip install https://github.com/dariobauer/graph-onedrive/archive/master.zip


## Documentation

### Authentication

Before the package can interact with the Graph API, it must first create an authenticated session with the Microsoft identity platform.

The Graph API operates securely through each request having an accompanying bearer token containing an "access token". These tokens are used to verify a session has been authenticated.
The package generates these access tokens on your behalf, acting as a native OAuth client.

For this process to work, you must configure the OAuth client.

#### Step 1: Create an Azure app

To interact with the Graph API, an app needs to be registered through the [Azure portal](https://portal.azure.com/). Detailed documentation on how to do this is [available directly from Microsoft](https://docs.microsoft.com/en-us/graph/auth-register-app-v2?context=graph%2Fapi%2F1.0&view=graph-rest-1.0).

Setup Option | Description
---|---
Supported account types | This is related to the tenant described in the next section. Essentially the more restrictive, the easier it is to get the app registered.
Redirect URI | It is recommended this is left as `Web` to `http://localhost`.

#### Step 2: Obtain authentication details

You need to obtain your registered app's authentication details from the [Azure portal](https://portal.azure.com/) to use in the package for authentication.

Parameter | Location within Azure Portal | Description
---|---|---
Directory (tenant) ID | App registrations > *your app name* > Overview | The tenant is used to restrict an app to certain accounts. `common` allows both personal Microsoft accounts as well as work/school accounts to use the app. `organizations`, and `consumers` each only allow these account types. All of these types are known as multi-tenant and require more security processes be passed to ensure that you are a legitimate developer. On the contrary, single-tenant apps restrict the app to only one work/school account (i.e. typically one company) and therefore have far fewer security requirements. Single tenants are either a GUID or domain. Refer to the [Azure docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-protocols#endpoints) for details.
Application (client) ID | App registrations > *your app name* > Overview | The application ID that's assigned to your app. You can find this information in the portal where you registered your app. Note that this is not the client secret id but the id of the app itself.
Client secret value | App registrations > *your app name* > Certificates & secrets | The client secret that you generated for your app in the app registration portal. Note that this allows you to set an expiry and should be checked if your app stops working. The client secret is hidden after the initial generation so it is important to copy it and keep it secure.

WARNING: The client secret presents a security risk if exposed. It is recommended to revoke the client secret immediately if it becomes exposed.

#### Step 3: Use the package to authenticate

##### a) Authenticate at the time of instance initialization

The easiest option is to just create an instance which will guide you through the authentication at the time of creating the instance. This is possible if you are running the script in the command line.

##### b) Generate a config file ahead of time

Create a config file before creating the instance is useful if your production code will not be run in a terminal.

You can preemptively create a config file using the command-line:

    graph-onedrive auth


### Creating an instance

#### Step 1: Import the package

Ensure the package is installed, refer to the installation section above.

    import graph_onedrive

#### Step 2: Create an instance

At the instance creation stage you must provide the authentication configuration.

The configuration can be either set within your code or a configuration file. It is recommended that you use the config file option and helper functions are provided to make this easier.

##### a) Using a config file (recommended)

The configuration file can be created for you using the command-line tool which will also authenticate at the same time. This is highly recommended:

    graph-onedrive auth

Alternatively, create a json file containing the following information:

    {
        "onedrive": {
            "tenant_id": "",
            "client_id": "",
            "redirect_url": "http://localhost:8080",
            "client_secret_value": ""
            }
        }
    }

Then create an instance of the Onedrive class:

    config_path = ""  # path to config.json
    my_instance = graph_onedrive.create_from_config_file(config_path)

Note that the `onedrive` dictionary key can be any string that is specified when creating the instance `create_from_config_file(config_path, config_key = "onedrive")`. This could facilitate multiple instances running from the same configuration file.

##### b) Using a set in-script config

This solution is slightly easier but could be a security issue, especially if sharing code.

    client_id = ""
    client_secret_value = ""
    tenant = ""
    redirect_url = "http://localhost:8080",
    my_instance = graph_onedrive.create(client_id, client_secret_value, tenant, redirect_url)


### Using refresh tokens

Access tokens used to authenticate each request directly by the Graph API, without having to re-authenticate each time with the authentication server which speeds up requests. However due to security, these tokens typically expire after one hour.
To avoid having to have the user re-authorize the app with their account, refresh tokens are used instead. This process of refreshing access tokens in managed automatically by the package, however if the script ends, then the instance information is lost.

To use the refresh token to create a new instance (e.g. after running the script again), you can save the refresh token and provide it at the time of instance initiation.

WARNING: Saving the refresh token presents a security risk as it could be used by anyone with it exchange it for an access token. If a refresh token is exposed, it is recommended that the app client secret it revoked. 

#### Obtaining the refresh token

##### a) Saving to a config file

If using a configuration file then you can have the helper functions save the refresh token to your configuration. Simply add the following call after your last API request.

    graph_onedrive.save_to_config_file(my_instance, config_path)

Note if using a dictionary key other than `onedrive`, you can specify it when saving as `save_to_config_file(... , config_key = "onedrive")`.

Then when creating the instance again using your config file, the refresh token will be used.

WARNING: This configuration assumes that your use of the configuration file serves one user only. If using the same code to serve multiple users then the refresh token must be stored independently of the configuration file.

##### b) Saving the token manually

You can get the refresh token from your instance:

    refresh_token = my_instance.refresh_token

Then when creating an instance later, provide the refresh token:

    my_instance = graph_onedrive.create(... , refresh_token = refresh_token)


### Making requests to the Graph API

These requests to the Graph API are made using the instance of the Onedrive class that you have created.

#### get_usage

Get the current usage and capacity of the connected Onedrive.

    used, capacity, units = my_instance.get_usage(unit, refresh, verbose)

Keyword arguments:

* unit (str) -- unit to return value, either "b", "kb", "mb", "gb" (default = "gb")
* refresh (bool) -- refresh the usage data (default = False)
* verbose (bool) -- print the usage (default = False)
            
Returns:

* used (float) -- storage used in unit requested
* capacity (float) -- storage capacity in unit requested
* units (str) -- unit of usage

#### list_directory

List the files and folders within the input folder/root of the connected Onedrive.

    items = my_instance.list_directory(folder_id, verbose)

Keyword arguments:

* folder_id (str) -- the item id of the folder to look into, None being the root directory (default = None)
* verbose (bool) -- print the items along with their ids (default = False)

Returns:

* items (dict) -- details of all the items within the requested directory

#### detail_item

Retrieves the metadata for an item.

    item_details = my_instance.detail_item(item_id, verbose)

Positional arguments:

* item_id (str) -- item id of the folder or file

Keyword arguments:

* verbose (bool) -- print the main parts of the item metadata (default = False)

Returns:

* items (dict) -- metadata of the requested item

#### make_folder

Creates a new folder within the input folder/root of the connected Onedrive.

    folder_id = my_instance.make_folder(folder_name, parent_folder_id, check_existing, if_exists)

Positional arguments:

* folder_name (str) -- the name of the new folder

Keyword arguments:

* parent_folder_id (str) -- the item id of the parent folder, None being the root directory (default = None)
* check_existing (bool) -- checks parent and returns folder_id if a matching folder already exists (default = True)
* if_exists (str) -- if check_existing is set to False; action to take if the new folder already exists, either "fail", "replace", "rename" (default = "rename")

Returns:

* folder_id (str) -- newly created folder item id

#### move_item

Moves an item (folder/file) within the connected Onedrive. Optionally rename an item at the same time.

    item_id, folder_id = my_instance.move_item(item_id, new_folder_id, new_name)

Positional arguments:

* item_id (str) -- item id of the folder or file to move
* new_folder_id (str) -- item id of the folder to shift the item to

Keyword arguments:

* new_name (str) -- optional new item name with extension (default = None)

Returns:

* item_id (str) -- item id of the folder or file that was moved, should match input item id
* folder_id (str) -- item id of the new parent folder, should match input folder id

#### copy_item

Copies an item (folder/file) within the connected Onedrive server-side.

    item_id = my_instance.copy_item(item_id, new_folder_id, new_name, confirm_complete)

Positional arguments:

* item_id (str) -- item id of the folder or file to copy
* new_folder_id (str) -- item id of the folder to copy the item to
* confirm_complete (bool) -- waits for the copy operation to finish before returning (default = False)

Keyword arguments:

* new_name (str) -- optional new item name with extension (default = None)

Returns:

* item_id (str) -- item id of the new item

#### rename_item

Renames an item (folder/file) without moving it within the connected Onedrive.

    item_name = my_instance.rename_item(item_id, new_name)

Positional arguments:

* item_id (str) -- item id of the folder or file to rename
* new_name (str) -- new item name with extension

Returns:

* item_name (str) -- new name of the folder or file that was renamed

#### delete_item

Deletes an item (folder/file) within the connected Onedrive. Potentially recoverable in the Onedrive web browser client.

    confirmation = my_instance.delete_item(item_id, pre_confirm)

Positional arguments:

* item_id (str) -- item id of the folder or file to be deleted

Keyword arguments:

* pre_confirm (bool) -- confirm that you want to delete the file and not show the warning (default = False)

Returns:

* confirmation (bool) -- True if item was deleted successfully

#### download_file

Downloads the file to the current working directory.
Note folders cannot be downloaded.

Future development improvement: specify the file location and name.

    file_name = my_instance.download_file(item_id)

Positional arguments:

* item_id (str) -- item id of the file to be deleted

Returns:

* file_name (str) -- returns the name of the file including extension

#### upload_file

Uploads a file to a particular folder with a provided file name.

    item_id = my_instance.upload_file(file_path, new_file_name, parent_folder_id, if_exists)

Positional arguments:

* file_path (str|Path) -- path of the file on the drive

Keyword arguments:

* new_file_name (str) -- new name of the file as it should appear on Onedrive, without extension (default = None)
* parent_folder_id (str) -- item id of the folder to put the file within, if None then root (default = None)
* if_exists (str) -- action to take if the new folder already exists, either "fail", "replace", "rename" (default = "rename")

Returns:

* item_id (str) -- item id of the newly uploaded file


## Examples

Examples are provided to aid in development: <https://github.com/dariobauer/graph-onedrive/blob/main/examples/>


## Links

* License: <https://github.com/dariobauer/graph-onedrive/blob/main/LICENSE>
* Change Log: <https://github.com/dariobauer/graph-onedrive/blob/main/CHANGES.md>
* PyPI Releases: <https://pypi.org/project/graph-onedrive>
* Source Code: <https://github.com/dariobauer/graph-onedrive/>
* Issue Tracker: <https://github.com/dariobauer/graph-onedrive/issues>