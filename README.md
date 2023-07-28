# DataGrid Advanced Client-Side Search

Building advanced client-side search forms for DataGrids


https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/5964bf79-e513-4227-9a72-361cee6c0d7b


<hr>

## Required setup
NOTE: These instructions apply to Stadium 6 only

### Database
Create the MSSQL database by running the SQL scripts in the Database folder of this repo. 

### DataGrid

1. Add a Database connector 
2. Add a query to select the data for the DataGrid
3. Add a DataGrid Control to a page
4. Define DataGrid columns in the DataGrid properties
5. Execute the query in the .Load event handler of the page
6. Assign the data returned by the query to the DataGrid using a SetValue function

### Filters form

1. Add a Grid control to the page 
2. Add a class entitled "filtergrid" to the grid control classes property
3. Add a button below the grid with the Text "Apply"

#### For Text columns

1. Add the following controls into a grid row
   1. A label control
   2. A DropDown control
   3. A TextBox control
2. Enter the name of the DataGrid column heading you want to filter into the label Text property
3. Add any or all of the operators shown below in any order as *values* for the DropDown options (assign any text to these options you like)
   1. Contains
   2. DoesNotContain
   3. Equals
   4. NotEquals

#### For Number columns

1. Add the following controls into a grid row
   1. A label control
   2. A DropDown control
   3. A Flexbox control
   4. Add two TextBox control next to each other into the Flexbox control (From and To)
2. Enter the name of the DataGrid column heading you want to filter into the label Text property
3. Add any or all of the operators shown below as *values* for the DropDown options (set "From-To" or "Between" as the first option)
   1. From-To
   2. Between
   3. GreaterThan
   4. SmallerThan

#### For Date columns

1. Add the following controls into a grid row
   1. A label control
   2. A DropDown control
   3. A Flexbox control
   4. Add two TextBox control next to each other into the Flexbox control (From and To)
2. Enter the name of the DataGrid column heading you want to filter into the label Text property
3. Add any or all of the operators shown below as *values* for the DropDown options (set "Between" as the first option)
   1. Between
   2. GreaterThan
   3. SmallerThan

#### For Boolean columns

1. Add the following controls into a grid row
   1. A label control
   2. A DropDown control
2. Enter the name of the DataGrid column heading you want to filter into the label Text property
3. Add any or all of the operators shown below as *values* for the DropDown options (set "ShowAll" as the first option)
   1. ShowAll
   2. Yes
   3. No

### Global scripts

Create six global scripts as per the attached sample application
1. ConstructSearchPhrase: Creates the SearchPhrase from the current phrase and the new phrase. Two decision actions evaluate whether to create a new phrase, add to an existing one or do nothing. 
1. ParseColumnHeading: Adds escape characters (\\) to spaces found in the column header. 
1. SetBooleanFilter: Creates a search prase for a boolean filter. 
1. SetDateFilter: Creates a search phrase for a date filter.
1. SetNumberFilter: Creates a search phrase for a number filter.
1. SetTextFilter: Creates a search phrase for a text filter. 

### Apply button event handler

1. Add a Click event handler for the "Apply" button
2. Add a variable called "SearchPhrase" inside the handler

For every filter in the grid
1. Add the global script that corresponds with the column data type (e.g. Text columns -> add SetTextFilter script)
2. Set the parameters required by the script
   1. ColumnHeading: The heading of the DataGrid column the filter needs to be applied to as it appears in the "Header Text" property of that column or in the Heading row of the rendered DataGrid (NOTE: DataGrid headings might contain spaces that the corresponding Database columns do not contain)
   2. Operator: The value of the "operator" DropDown of the filter
   3. Values: 
      1. For Text fields: value of the TextBox
      2. For Number and Date Fields: The values of the From and To TextBoxes
      3. For Booleans: The value of the DropDown control of the filter
      4. SearchPhrase: The variable entitled SearchPhrase
   4. Under each call, add a SetValue control and assign the reault of the script to the "SearchPhrase" variable 
3. After all script calls, add a SetValue control and assign the "SearchPhrase" to the "SearchTerm" property of the DataGrid

### For each Number and Date filter

1. Create a "Change" event handler for the operator DropDown control
2. Add a Decision into the Handler
   1. If the value of the DropDown of this filter is "SmallerThan" or "GreaterThan" set the "To" TextBox visibility to false
   2. Else  set the "To" TextBox visibility to true

<hr>

## Optional setup

### Clear button

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
