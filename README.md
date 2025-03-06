# Azure Bulk Import File Generator

The main goal for this script is to generate Pulumi code for existing Azure resources.
Although it uses import to bring the resources under Pulumi management, the main goal is to generate code.
So, the assumption is that the state file will be deleted at the end of the process.

Pulumi supports [importing](https://www.pulumi.com/docs/iac/adopting-pulumi/import/) existing resources.
The import process does two things:
- Brings the resources into Pulumi state management, and
- Generates code for the imported resources.

The script in this repo generates [bulk import](https://www.pulumi.com/docs/iac/adopting-pulumi/import/#bulk-import-operations) files based on the output of `az resource list`.

# General Import Process

* Be sure you are logged into the Azure CLI.
  * e.g. `az login`
* If necessary, initialize a stack into which the resources will be imported.
  * Create a folder for the Pulumi project.
  * Use pulumi new to initialize the stack.
    * e.g. `pulumi new azure-csharp` 
* Run the `gen_bulk_import_file` script (see [Usage](#Usage)).
* For each generated bulk import file do the following:
  * `pulumi import -f BULK_IMPORT_FILE_NAME -o GEN_CODE`
    * Where `BULK_IMPORT_FILE_NAME` is the name of the bulk import file you are processing.
    * Where `GEN_CODE` is the name of a file into which the generated code will be placed.
  * Edit the generated code to clean it up.
    * Remove any default property settings.
      * This cleans up the code by removing unnecessary lines.
      * It is recommended to just comment out lines for the time being until the code is validated below.
      * You can generally guess which properties are defaults if they are set to false or null values.
      * A later step will show you if you were overzealous with your removal and can put things back (hence the commenting out for now).
    * Change any hard-coded references to references to other resource's properties.
      * You may do this at the very end once all the resources are in the program.
      * Pulumi AI/Copilot can help do this abstraction as well.
        * e.g. Ask it to "replace hard-coded values with resource property references for the following program" and pass the program.
  * Copy the code from the generated code file into the main Pulumi program file.
  * Run `pulumi preview` to validate the code matches the state.
    * The preview should show NO changes.
    * If any changes are detected, remove comments for the properties that are showing changes.
    * You may also want to put repeated resource declarations into a loop to make the code DRYer.
    * You can put code into functions if desired but the final steps will look for other ways of creating reusable constructs.
    * Repeat the process until the preview shows no changes.
* Once done with all the imports, run one more `pulumi preview` to confirm the code matches the state.
* The state can now be deleted (without affecting any of the resources). This enables one to then put the final touches on the code.
  * `pulumi stack rm --force` will delete the state (without touching the actual resources)
  * If you want to keep the resources managed by pulumi, then don't do this but some of the final cleaning steps below may not be possible without using the [aliases](https://www.pulumi.com/docs/iac/concepts/options/aliases/) resource option.  
* Final cleaning of the program code (ONLY DO IF YOU DELETED THE STATE or you understand the implications of these changes).
  * The generated code uses rather simple names for the resource declarations. So you'll likely want to clean up the code in this regard.
    * E.g. `storageaccount1 = ...` should probably be something like `appstorage = ...`
  * Make the program reusable by implementing [stack configuration](https://www.pulumi.com/docs/iac/concepts/config/).
    * The goal is to allow the same program to be used to deploy multiple, separate environments.
    * The code probably has various hard-coded values that you want to abstract on a stack-by-stack basis.
  * Put any reusable code into loops, functions and/or [component resources](https://www.pulumi.com/docs/iac/concepts/resources/components/) 
    * The goal is to make the code as readable and maintanable as possible by leveraging language constructs like loops, functions, classes, etc.
* Initialize a new stack and deploy it.
  * `pulumi stack init test`
  * `pulumi up`

# Usage

* Generate a json file containing the list of resources in azure.
  * e.g. `az resource list --resource-group RESOURCE_GROUP_NAME > RESOURCE_LIST_FILE`
* Run the `gen_bulk_import_file` script.
  * `python gen_bulk_import_file -h` for list of parameters.
  * E.g. `python gen_bulk_import_file -r az_resources.json` 
  * See below for what to do if a "missing Pulumi type" error is seen.
* One or more "pulumi bulk import" files will be generated.
  * By default, five resources will be in each import file to make the import process (see below) a bit easier to manage.
  * The number of resources in each import file can be modified by passing the `-n` option.

## Missing Pulumi Type Error Handling

If the script throws an error about not being able to find a Pulumi type in `type-mappings.json` then add an entry similar the others in the file.

You can use the Pulumi [Azure-Native provider](https://www.pulumi.com/registry/packages/azure-native/) documentation to find the matching Pulumi type as follows:
* Navigate to the docs page for the resource in question. For example, the docs page for [resource group](https://www.pulumi.com/registry/packages/azure-native/api-docs/resources/resourcegroup/).
* On the resource's docs page, navigate to the "Import" section. For example, [resource group import](https://www.pulumi.com/registry/packages/azure-native/api-docs/resources/resourcegroup/#import).
* Looking at the provided import example, you will see the Pulumi type. For example, for reaource group: `azure-native:resources:ResourceGroup`

