# DataGrid Advanced Client-Side Search

Building advanced client-side search forms for DataGrids


https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/5964bf79-e513-4227-9a72-361cee6c0d7b


<hr>

## Version
1.1

### Changes
Converted 'Boolean Filter' to a generic 'Enum Filter'. This filter still works for Boolean columns, but can now also be applied to any other column

## Sample applications
This repo contains two Stadium 6 applications. 
1. [AdvancedClientSideSearchForm.sapz](Stadium6/AdvancedClientSideSearchForm.sapz?raw=true)
Created using Stadium actions (easier to understand)
2. [AdvancedClientSideSearchForm_JS.sapz](Stadium6/AdvancedClientSideSearchForm_JS.sapz?raw=true)
Created using Javascript actions (easier to recreate)

<hr>

## Setup applicability
These instructions apply to Stadium 6 only

<hr>

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Database, Connector and DataGrid
Use the instructions from [this repo](https://github.com/stadium-software/samples-database) to setup the database and DataGrid for this sample

<hr>

## Filters form

1. Add a Grid control to the page and place it above the DataGrid
2. Add a class entitled "filtergrid" to the grid control classes property
3. Add a button below the grid with the Text "Apply"
4. Place controls into the Grid control as shown below

![Form-Controls](https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/35ebab95-12e2-4416-8aaf-bd302c2f6462)

### For Text columns

1. Enter the name of the filter into the Label Text property (e.g. FirstName)
2. Add any or all of the operators shown below in any order as *values* for the DropDown  *Options* property (assign any text to these options you like)
   1. Contains
   2. DoesNotContain
   3. Equals
   4. NotEquals

### For Number columns

1. Enter the name of the filter into the Label Text property
2. Add any or all of the operators shown below as *values* for the DropDown  *Options* property (set "From-To" or "Between" as the first option)
   1. From-To
   2. Between
   3. GreaterThan
   4. SmallerThan

### For Date columns

1. Enter the name of the filter into the Label Text property
2. Add any or all of the operators shown below as *values* for the DropDown  *Options* property (set "Between" as the first option)
   1. Between
   2. GreaterThan
   3. SmallerThan

### Enum Filter

1. Enter the name of the filter into the label Text property
2. Add "ShowAll" as the first value in the filter and any number of text values you want people to select from as *values* for the DropDown *Options* property (e.g. Yes / No or a list of statuses). 

<hr>

## Global scripts

Create six global scripts listed below as per the attached sample application. 

NOTES: If you are using Stadium 6.6 or later, you can copy the global scripts from the [sample application](Stadium6/AdvancedClientSideSearchForm_JS.sapz?raw=true) into your own. 

1. ConstructSearchPhrase: Creates the SearchPhrase from the current phrase and the new phrase. Two decision actions evaluate whether to create a new phrase, add to an existing one or do nothing. 
1. ParseColumnHeading: Adds escape characters (\\) to spaces found in the column header. 
1. SetEnumFilter: Creates a search phrase for a value selected from a dropdown. 
1. SetDateFilter: Creates a search phrase for a date filter.
1. SetNumberFilter: Creates a search phrase for a number filter.
1. SetTextFilter: Creates a search phrase for a text filter. 

<hr>

## Apply button event handler

1. Add a *Click* event handler for the "Apply" button
2. Add a *Variable* called "SearchPhrase" inside the handler
-------------------
For every filter in the grid
1. Add the global script that corresponds with the column data type 
   1. Text columns: add *SetTextFilter* script
   2. Date columns: add *SetDateFilter* script
   3. Number columns: add *SetNumberFilter* script
   4. Boolean columns:  add *SetBooleanFilter* script
2. Set the parameters required by the script
   1. ColumnHeading: The heading of the DataGrid column the filter needs to be applied to as it appears in the "Header Text" property of that column or in the Heading row of the rendered DataGrid (NOTE: DataGrid headings might contain spaces that the corresponding Database columns do not contain)
   2. Operator: The value of the "operator" DropDown of the filter
   3. Values: 
      1. For Text fields: value of the TextBox
      2. For Number and Date Fields: The values of the From and To TextBoxes
      3. For Enums: The value of the DropDown control of the filter
      4. SearchPhrase: The variable entitled SearchPhrase
![ApplyButtonSetup](https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/053ce56b-06c6-42be-b76f-7e505f67cf66)

   1. Under each call, add a SetValue control and assign the reault of the script to the "SearchPhrase" variable
![SetSearchPhrase](https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/432e2f33-17d0-4aca-b37f-952af5ffcfa4)
-------------------
3. After all script calls, add a SetValue control and assign the "SearchPhrase" to the "SearchTerm" property of the DataGrid
![SetDGSearchTerm](https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/0837b0f8-168f-4524-b5d2-f1a3f82e76cd)

## For each Number and Date filter

1. Create a "Change" event handler for the operator DropDown control
2. Add a Decision into the Handler
   1. If the value of the DropDown of this filter is "SmallerThan" or "GreaterThan" set the "To" TextBox visibility to false
   2. Else  set the "To" TextBox visibility to true

<hr>

# Optional setup

## Clear button

1. Add a script to the page called "ClearForm"
2. Add a SetValue into the handler and set the DataGrid SearchTerm property to empty (= "")
3. Add a JavaScript action to the handler with the following JS code

```
const pageName = window.location.pathname.replace("/", "");
let arr = document.querySelectorAll(".filtergrid input");
for (let i=0;i<arr.length;i++){
 let inputName = arr[i].id.replace(`${pageName}_`, "");
 this[`${inputName}Text`] = "";
}
arr = document.querySelectorAll(".filtergrid select");
for (let i=0;i<arr.length;i++){
 let inputName = arr[i].id.replace(`${pageName}_`, "");
 
  if (this[`${inputName}Options`][0].value == "ShowAll"){
  this[`${inputName}SelectedValue`] = undefined;
 } 
}
```

4. Add a button with the text "Clear" to the page
5. Create the Click event handler for the button
6. In the handler, call the ClearForm script
7. Drag a JavaScript action to the Apply button click event handler with the following JS code

```
let th = this;
let clearfilter = document.querySelector(".clear-filter");
clearfilter.addEventListener("click", function () {
 th.ClearForm();
});
```

# Optional Custom Development
1. Allowing users to save searches to the database (save generated search phase)
2. Providing a dropdown for users to apply previously saved searches (populate search field with previously saved phrase)
