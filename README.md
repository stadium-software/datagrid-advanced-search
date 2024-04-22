# DataGrid Client Side Filters

Building advanced client-side search forms for DataGrids

https://github.com/stadium-software/datagrid-advanced-search/assets/2085324/a3a601aa-1040-44f1-9beb-cf56af1ad9d3

## Version
2.4

### Change Log
2.0 Complete rewrite of the feature. Simplified setup by generating all form elements in JS script. Added [display modes](#display-modes) (standard, collapsed and integrated)

2.1 Fixed selectable column bug

2.2 Changed Save/Apply button text

2.3 Added version to CSS; fixed url parsing bug

2.4 Switched column parameter from [heading property to name property](#pageload-setup); fixed number and date input display bug; general JS cleanup

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Database, Connector and DataGrid
Use the instructions from [this repo](https://github.com/stadium-software/samples-database) to setup the database and DataGrid for this sample

## Global Script Setup
1. Create a Global Script and name it "DataGridFilter"
2. Add the input parameters below to the script
   1. DataGridClass
   2. DisplayMode
   3. FilterConfig
   4. FilterContainerClass
   5. FilterHeading
   6. CollapseOnClickAway
3. Drag a Javascript action into the script and paste the Javascript below unaltered into the action
```javascript
/* Stadium Script v2.4 https://github.com/stadium-software/datagrid-advanced-search */
let scope = this;
let filterClassName = "." + ~.Parameters.Input.FilterContainerClass;
let dgClassName = "." + ~.Parameters.Input.DataGridClass;
let filterConfig = ~.Parameters.Input.FilterConfig;
let displayMode = ~.Parameters.Input.DisplayMode;
if (displayMode) displayMode = displayMode.toLowerCase();
let arrPageName = window.location.pathname.split("/");
let pageName = arrPageName[arrPageName.length - 1];
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    dg = document.querySelector(".data-grid-container");
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
} else { 
    dg = dg[0];
}
dg.classList.add("stadium-filtered-datagrid");
let datagridname = dg.id.replace(`${pageName}_`, "").replace("-container","");
let dataGridColumns = getColumnDefinition();
let filterContainer = document.querySelectorAll(filterClassName);
if (filterContainer.length == 0) {
    console.error("The container for the filter was not found. Drag a container control into the page and assign the class '" + filterClassName + "' to it.");
    return false;
} else if (dg.length > 1) {
    console.error("The class '" + filterClassName + "' is assigned to multiple controls. Assign a unique classname to the filter container");
    return false;
} else { 
    filterContainer = filterContainer[0];
}
filterContainer.classList.add("stadium-filter-container");
let filterInnerContainer = document.createElement("div");
filterInnerContainer.classList.add("stadium-filter-inner-container");
filterContainer.appendChild(filterInnerContainer);
let stadiumFilters = document.createElement("div");
stadiumFilters.classList.add("stadium-filters");
filterInnerContainer.appendChild(stadiumFilters);

if (displayMode == "integrated" || displayMode == "collapsed") {
    let filterHeader = document.createElement("div");
    filterHeader.classList.add("stadium-filter-header");
    filterInnerContainer.before(filterHeader);
    filterHeader.addEventListener("click", function (e) {
        e.target.closest(".stadium-filter-container").classList.toggle("expand");
    });
    if (displayMode == "integrated") { 
        filterContainer.classList.add("filter-integrated");
        let datagridheader = dg.querySelector(".data-grid-header");
        if (datagridheader.querySelector("div")) {
            datagridheader.querySelector("div:nth-child(1)").before(filterContainer);
        } else { 
            datagridheader.appendChild(filterContainer);
        }
    } else if (displayMode == "collapsed") { 
        filterHeader.textContent = "Advanced Filter";
        filterContainer.classList.add("filter-collapsed");
    }
}
const insert = (arr, index, newItem) => [...arr.slice(0, index), newItem, ...arr.slice(index)];
let numberSelectChange = (e) => {
    let target = e.target;
    let toEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-to-number");
    let fromEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-from-number");
    if (target.value != "Between" && target.value != "From-To") {
        toEl.classList.add("visually-hidden");
        fromEl.setAttribute("placeholder", "Value");
    } else { 
        toEl.classList.remove("visually-hidden");
        fromEl.setAttribute("placeholder", "From value");
    }
};
let dateSelectChange = (e) => {
    let target = e.target;
    let toEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-to-date");
    let fromEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-from-date");
    if (target.value == "Greater than" || target.value == "Smaller than") {
        toEl.classList.add("visually-hidden");
        fromEl.setAttribute("placeholder", "Value");
    } else { 
        toEl.classList.remove("visually-hidden");
        fromEl.setAttribute("placeholder", "From value");
    }
};

initFilterForm();

function initFilterForm() {
    for (let i = 0; i < filterConfig.length; i++) {
        let column = filterConfig[i].column;
        let columnDef, colNo;
        if (!isNumber(column)) {
            columnDef = getElementFromObjects(dataGridColumns, column, "name");
            if (!columnDef) {
                console.error("Column '" + column + "' was not found. The 'column' property must contain the column name exactly as defined in the DataGrid 'Columns' property or the column number.");
                continue;
            }
            colNo = dataGridColumns.map(function (e) {return e.name;}).indexOf(columnDef.name) + 1;
        } else {
            colNo = column;
            columnDef = dataGridColumns[column - 1];
            column = columnDef.name;
        }
        if (!colNo || !columnDef.headerText) {
            if (!columnDef.headerText) {
                console.error("Column '" + columnDef.name + "' has no header text. Filter columns must have a header text.");
            }
            continue;
        }
        let type = filterConfig[i].type;
        let name = filterConfig[i].name;
        let data = filterConfig[i].data;
        let display = filterConfig[i].display;
        
        let label = document.createElement("div");
        label.classList.add("control-container","label-container");
        let labelInner = document.createElement("span");
        labelInner.textContent = name;
        label.appendChild(labelInner);

        let operator = document.createElement("div");
        let valueField = document.createElement("div");
        let select, input;

        if (type == "text") {
            select = document.createElement("select");
            let options = ["Contains", "Does Not Contain", "Equals", "Does Not Equal"];
            for(let s = 0; s < options.length; s++) {
                let opt = options[s];
                let el = document.createElement("option");
                el.textContent = opt;
                el.value = opt;
                select.appendChild(el);
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            input = document.createElement("input");
            input.classList.add("form-control", "text-box-input", "filtergrid-text-value");
            input.setAttribute("placeholder", "Text");
            valueField.classList.add("control-container","text-box-container");
        }
        if (type == "number") {
            select = document.createElement("select");
            let options = ["Between", "From-To", "Greater than", "Smaller than"];
            for(let s = 0; s < options.length; s++) {
                let opt = options[s];
                let el = document.createElement("option");
                el.textContent = opt;
                el.value = opt;
                select.appendChild(el);
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            let numInput1 = document.createElement("input");
            numInput1.classList.add("form-control", "text-box-input", "filtergrid-from-number");
            numInput1.setAttribute("placeholder", "From value");
            let numInput2 = document.createElement("input");
            numInput2.classList.add("form-control", "text-box-input", "filtergrid-to-number");
            numInput2.setAttribute("placeholder", "To value");
            input = document.createElement("div");
            input.classList.add("number-values");
            select.addEventListener("change", numberSelectChange);
            input.appendChild(numInput1);
            input.appendChild(numInput2);
        }
        if (type == "date") {
            select = document.createElement("select");
            let options = ["Between", "Greater than", "Smaller than"];
            for(let s = 0; s < options.length; s++) {
                let opt = options[s];
                let el = document.createElement("option");
                el.textContent = opt;
                el.value = opt;
                select.appendChild(el);
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            let dtInput1 = document.createElement("input");
            dtInput1.classList.add("form-control", "text-box-input", "filtergrid-from-date");
            dtInput1.setAttribute("placeholder", "From date");
            let dtInput2 = document.createElement("input");
            dtInput2.classList.add("form-control", "text-box-input", "filtergrid-to-date");
            dtInput2.setAttribute("placeholder", "To date");
            input = document.createElement("div");
            input.classList.add("date-values");
            select.addEventListener("change", dateSelectChange);
            input.appendChild(dtInput1);
            input.appendChild(dtInput2);
        }
        if (type == "boolean") {
            let options = ["Show all", "Yes", "No"];
            if (display == "radio") {
                select = document.createElement("div");
                for (let s = 0; s < options.length; s++) {
                    let cont = document.createElement("div");
                    cont.classList.add("radio");
                    let opt = options[s];
                    let el = document.createElement("input");
                    let lab = document.createElement("label");
                    el.type = "radio";
                    el.name = column;
                    el.checked = false;
                    if (opt == "Show all") el.checked = true;
                    el.value = opt;
                    lab.textContent = opt;
                    let fid = column + "_" + opt;
                    el.id = fid.replaceAll(" ", "").toLowerCase();
                    lab.setAttribute("for", el.id);
                    cont.appendChild(el);
                    cont.appendChild(lab);
                    select.appendChild(cont);
                }
                operator.classList.add("control-container", "radio-button-list-container", "filtergrid-radiobutton-list");
            } else {
                select = document.createElement("select");
                for(let s = 0; s < options.length; s++) {
                    let opt = options[s];
                    let el = document.createElement("option");
                    el.textContent = opt;
                    el.value = opt;
                    select.appendChild(el);
                }
                select.classList.add("form-control");
                operator.classList.add("control-container", "drop-down-container", "filtergrid-boolean-operator");
            }
            input = document.createElement("div");
        }
        if (type == "enum") {
            data = insert(data, 0, "Show all");
            if (display == "radio") {
                select = document.createElement("div");
                for (let s = 0; s < data.length; s++) {
                    let cont = document.createElement("div");
                    cont.classList.add("radio");
                    let opt = data[s];
                    let el = document.createElement("input");
                    let lab = document.createElement("label");
                    el.type = "radio";
                    el.name = column;
                    el.checked = false;
                    if (opt == "Show all") el.checked = true;
                    el.value = opt;
                    lab.textContent = opt;
                    let fid = column + "_" + opt;
                    el.id = fid.replaceAll(" ", "").toLowerCase();
                    lab.setAttribute("for", el.id);
                    cont.appendChild(el);
                    cont.appendChild(lab);
                    select.appendChild(cont);
                }
                operator.classList.add("control-container", "radio-button-list-container", "filtergrid-radiobutton-list");
            } else {
                select = document.createElement("select");
                for (let s = 0; s < data.length; s++) {
                    let opt = data[s];
                    let el = document.createElement("option");
                    el.textContent = opt;
                    el.value = opt;
                    select.appendChild(el);
                }
                select.classList.add("form-control");
                operator.classList.add("control-container", "drop-down-container", "filtergrid-enum-operator");
            }
            input = document.createElement("div");
        }
        if (type == "multiselect") {
            select = document.createElement("div");
            for (let s = 0; s < data.length; s++) {
                let cont = document.createElement("div");
                cont.classList.add("checkbox");
                let opt = data[s];
                let el = document.createElement("input");
                let lab = document.createElement("label");
                el.type = "checkbox";
                el.checked = false;
                el.value = opt;
                lab.textContent = opt;
                let fid = column + "_" + opt;
                el.id = fid.replaceAll(" ", "").toLowerCase();
                lab.setAttribute("for", el.id);
                cont.appendChild(el);
                cont.appendChild(lab);
                select.appendChild(cont);
            }
            operator.classList.add("control-container", "check-box-list-container", "filtergrid-checkbox-list");
            input = document.createElement("div");
        }
        setAttributes(operator, { "foperator": column, "ftype": type, "cno": colNo, "fdisplay": display });
        operator.appendChild(select);

        setAttributes(valueField, { "fvalue": column, "ftype": type, "cno": colNo, "fdisplay": display });
        valueField.appendChild(input);

        stadiumFilters.appendChild(label);
        stadiumFilters.appendChild(operator);
        stadiumFilters.appendChild(valueField);
    }
    let buttonBar = document.createElement("div"); buttonBar.classList.add("filter-button-bar");

    let clearButton = document.createElement("button");
    clearButton.textContent = "Clear";
    clearButton.classList.add("lite-button", "btn", "btn-lg", "btn-default");
    clearButton.addEventListener("click", clearForm);
    let clearButtonContainer = document.createElement("div");
    clearButtonContainer.classList.add("control-container", "button-container");
    clearButtonContainer.appendChild(clearButton);
    buttonBar.appendChild(clearButtonContainer);

    let saveButton = document.createElement("button");
    saveButton.textContent = "Apply";
    saveButton.classList.add("btn", "btn-lg", "btn-default");
    saveButton.addEventListener("click", filterDataGrid);
    let saveButtonContainer = document.createElement("div");
    saveButtonContainer.classList.add("control-container", "button-container");
    saveButtonContainer.appendChild(saveButton);
    buttonBar.appendChild(saveButtonContainer);

    stadiumFilters.appendChild(buttonBar);
    document.body.addEventListener("click", function(e){
        if (!e.target.closest(filterClassName)) {
            let allFilters = document.querySelectorAll(filterClassName);
            for (let i=0;i<allFilters.length;i++){
                allFilters[i].classList.remove("expand");
            }
        }
    });
}

function filterDataGrid() {
    let searchPhrase = [];
    let operatorEls = stadiumFilters.querySelectorAll("[foperator]");
    for (let i = 0; i < operatorEls.length; i++) {
        let ftype = operatorEls[i].getAttribute("ftype");
        let colNo = operatorEls[i].getAttribute("cno");
        let fdisplay = operatorEls[i].getAttribute("fdisplay");
        let colText = dataGridColumns[colNo - 1].headerText;
        let fvalueEl = operatorEls[i].nextElementSibling;
        let output, heading = "";
        if (colText) heading = colText.replaceAll(" ", "\\ ");
        if (ftype == "text") {
            let txtoperator = operatorEls[i].querySelector("select").value;
            let txtvalue = fvalueEl.querySelector("input").value;
            if (txtoperator == "Contains" && txtvalue) {
                output = heading + ':"' + txtvalue + '"';
            } else if (txtoperator == "Does Not Contain" && txtvalue) {
                output = heading + ':(NOT "' + txtvalue + '")';
            } else if (txtoperator == "Equals" && txtvalue) {
                output = heading + ':["' + txtvalue + '" TO "' + txtvalue + '"]';
            } else if (txtoperator == "Does Not Equal" && txtvalue) {
                output = heading + ':(NOT ["' + txtvalue + '" TO "' + txtvalue + '"])';
            }
        }
        if (ftype == "number") {
            let numoperator = operatorEls[i].querySelector("select").value;
            let numvaluefrom = fvalueEl.querySelector(".filtergrid-from-number").value;
            let numvalueto = fvalueEl.querySelector(".filtergrid-to-number").value;
            if (numvaluefrom) {
                if (numoperator.toLowerCase() == "between") {
                    output = heading + ':{' + numvaluefrom + ' TO ' + numvalueto + '}';
                } else if (numoperator.toLowerCase() == "from-to") {
                    output = heading + ':[' + numvaluefrom + ' TO ' + numvalueto + ']';
                } else if (numoperator.toLowerCase() == "greater than") {
                    output = heading + ':{' + numvaluefrom + ' TO 9007199254740991}';
                } else if (numoperator.toLowerCase() == "smaller than") {
                    output = heading + ':{-9007199254740991 TO ' + numvaluefrom + '}';
                }
            }
        }
        if (ftype == "date") {
            let dtoperator = operatorEls[i].querySelector("select").value;
            let dtvaluefrom = fvalueEl.querySelector(".filtergrid-from-date").value;
            let dtvalueto = fvalueEl.querySelector(".filtergrid-to-date").value;
            if (dtvaluefrom) {
                if (dtoperator.toLowerCase() == "between") {
                    output = heading + ':{' + dtvaluefrom + ' TO ' + dtvalueto + '}';
                } else if (dtoperator.toLowerCase() == "greater than") {
                    output = heading + ':{' + dtvaluefrom + ' TO 3000/01/01}';
                } else if (dtoperator.toLowerCase() == "smaller than") {
                    output = heading + ':{1000/01/01 TO ' + dtvaluefrom + '}';
                }
            }
        }
        if (ftype == "boolean" && fdisplay == "radio") {
            let multioperator = operatorEls[i].querySelectorAll("input[type='radio']");
            for (let s = 0; s < multioperator.length; s++) {
                if (multioperator[s].checked && multioperator[s].value != "Show all") {
                    output = heading + ':' + multioperator[s].value;
                }
            }
        } else if (ftype == "boolean") {
            let booloperator = operatorEls[i].querySelector("select").value;
            if (booloperator != "Show all" && booloperator != "") {
                output = heading + ':' + booloperator;
            }
        }
        if (ftype == "enum" && fdisplay == "radio") {
            let multioperator = operatorEls[i].querySelectorAll("input[type='radio']");
            for (let s = 0; s < multioperator.length; s++) {
                if (multioperator[s].checked && multioperator[s].value != "Show all") {
                    output = heading + ':["' + multioperator[s].value + '" TO "' + multioperator[s].value + '"]';
                }
            }
        } else if (ftype == "enum") {
            let enumoperator = operatorEls[i].querySelector("select").value;
            if (enumoperator != "Show all" && enumoperator != "") {
                output = heading + ':["' + enumoperator + '" TO "' + enumoperator + '"]';
            }
        }
        if (ftype == "multiselect") {
            let multioperator = operatorEls[i].querySelectorAll("input[type='checkbox']");
            let or = "";
            output = "(";
            for (let s = 0; s < multioperator.length; s++) {
                if (multioperator[s].checked) {
                    output += or + heading + ':["' + multioperator[s].value + '" TO "' + multioperator[s].value + '"]';
                    or = " OR ";
                }
            }
            output += ")";
        }
        if (output && output != "()") searchPhrase.push(output);
    }
    filterContainer.classList.remove("expand");
    scope[`${datagridname}SearchTerm`] = searchPhrase.join(' AND ');
}
function clearForm() { 
    let allCheckboxes = stadiumFilters.querySelectorAll("input[type='checkbox']");
    for (let i = 0; i < allCheckboxes.length; i++) {
        allCheckboxes[i].checked = false;
    } 
    let allRadios = stadiumFilters.querySelectorAll("input[type='radio']");
    for (let i = 0; i < allRadios.length; i++) {
        allRadios[i].checked = false;
        if (allRadios[i].value == "Show all") allRadios[i].checked = true;
    } 
    let allInputs = stadiumFilters.querySelectorAll("input:not([type='checkbox'],[type='radio'])");
    for (let i = 0; i < allInputs.length; i++) {
        allInputs[i].value = "";
    }   
    let allSelects = stadiumFilters.querySelectorAll("select");
    for (let i = 0; i < allSelects.length; i++) {
        allSelects[i].options.selectedIndex = 0;
    }
    let visuallyHidden = stadiumFilters.querySelectorAll(".visually-hidden");
    for (let i = 0; i < visuallyHidden.length; i++) {
        visuallyHidden[i].classList.remove("visually-hidden");
    }
    scope[`${datagridname}SearchTerm`] = null;
}
function setAttributes(el, attrs) {
  for(var key in attrs) {
    el.setAttribute(key, attrs[key]);
  }
}
function getColumnDefinition() {
    let colDefs = scope[`${datagridname}ColumnDefinitions`];
    if (scope[`${datagridname}DataGridHasSelectableData`]) {
        colDefs.unshift({name:"RowSelector", headerText: "RowSelector"});
    }
    return colDefs;
}
function isNumber(str) {
    if (typeof str == "number") return true;
    return !isNaN(str) && !isNaN(parseFloat(str));
}
function getElementFromObjects(haystack, needle, column) {
    return haystack.find(obj => {return obj[column] == needle;});
}
```

## Type Setup
1. Add a type called "FilterConfig" to the types collection in the Stadium Application Explorer
2. Add the following properties to the type
   1. type (Any)
   2. name (Any)
   3. column (Any)
   4. display (Any)
   5. data (List)
      1. Item (Any)

![Type Setup](images/TypeSetup.png)

## Page Setup
1. Add a *Container* control to the page
2. Add a class to uniquely identify the *Container* control to the control classes property (e.g. filter-container)
3. Add the *DataGrid* control the filter must apply to to the page 
4. Add a class to uniquely identify the *DataGrid* control to the control classes property (e.g. filter-datagrid)

![Page Setup](images/PageSetup.png)

## Page.Load Setup
1. Populate the *DataGrid* control with data by dragging on a query and assigning it using a *SetValue* (see [this repo](https://github.com/stadium-software/samples-database))
2. Drag a *List* action under the *SetValue*
3. Assign the *FilterConfig* type to the *List* action
4. Define the fields in the filtergrid
   1. *type*: the data type of the column you wish to enable filtering for
      1. text
      2. date
      3. number
      4. boolean (dropdown or radiobuttonlist)
      5. enum (dropdown or radiobuttonlist)
      6. multiselect (checkboxlist)
   2. *name*: the label displayed for the filter
   3. *column*: the number of the DataGrid column or the column name the filter must be applied to as specified in the DataGrid "Column" property
![](images/ColumnPropertyName.png)
   4. *display*: applies to filters of type *boolean* and *enum* only. These are shown as dropdowns by default, but passing the value "radio" in this property will cause them to be shown as radio button lists instead
   5. *data*: filters of type *enum* and *multiselect* require a list of data users can select from

Fields Definition Example
```json
= [{
	"type": "text",
	"name": "First Name",
	"column": "FirstName"
},{
    "type": "text",
    "name": "Last Name",
    "column": 3
},{
	"type": "date",
	"name": "Start Date",
	"column": "StartDate"
},{
	"type": "date",
	"name": "End Date",
	"column": "EndDate"
},{
	"type": "number",
	"name": "Number Of Pets",
	"column": "NoOfPets"
},{
	"type": "boolean",
	"name": "Healthy",
	"column": "Healthy",
	"display": "radio"
},{
	"type": "boolean",
	"name": "Happy",
	"column": "Happy",
	"display": "radio"
},{
	"type": "enum",
	"name": "Number of Children",
	"column": "NoOfChildren",
	"data": [0,1,2,3,4,5,6,7,8,9,10]
},{
	"type": "multiselect",
	"name": "Subscription",
	"column": "Subscription",
	"data": ["No data","Subscribed","Unsubscribed"]
}]
```
5. Drag the "DataGridFilter" global script below the *List*
6. Enter parameters for the script
   1. DataGridClass: The unique classname you assigned to the DataGrid above (e.g. filter-datagrid)
   2. DisplayMode: This parameter accepts three values
      1. Empty (default)
      2. "collapsed"
      3. "integrated"
   3. FilterConfig: Select the List containing the filter configurations you created from the dropdown
   4. FilterContainerClass: The unique classname you assigned to the Container control above (e.g. filter-container)
   5. FilterHeading: The title of the filter (not used in "integrated" display mode)
   6. CollapseOnClickAway (true / false): Applies only when Filters are shown in "collapsed" or "integrated" display mode. Defines whether to collapse the filter container when the user clicks elsewhere on the page. Default is "true"

![Script Parameters Example](images/ScriptParametersExample.png)

## Display Modes
1. Leaving this parameter empty means the filter is showing as a block wherever the *Container* control is placed on the page

![Standard Display](images/Standard.gif)

2. Adding the word "collapsed" in the parameter causes the filter to be expandable by the user. When expanded, the filter will displace any controls below it on the page

![Collapsed Display](images/Collapsed.gif)

3. Adding the word "integrated" in the parameter causes the filter to be expandable and shown as an icon next to the search bar. When it is opened, the filter will overlay other controls onthe page

![Integrated Display](images/Integrated.gif)

## Applying the CSS
The CSS below is required for the correct functioning of the module. Some elements can be [customised](#customising-css) using a variables CSS file. 

**Stadium 6.6 or higher**
1. Create a folder called "CSS" inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-custom-filters-variables.css*](datagrid-custom-filters-variables.css) and [*datagrid-custom-filters.css*](datagrid-custom-filters.css) into that folder
3. Paste the link tags below into the *head* property of your application
```html
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-custom-filters.css">
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-custom-filters-variables.css">
``` 

![](images/ApplicationHeader.png)

**Versions lower than 6.6**
1. Copy the CSS from the two css files into the Stylesheet in your application

## Customising CSS
1. Open the CSS file called [*datagrid-custom-filters-variables.css*](datagrid-custom-filters-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit
3. Overwrite the file in the CSS folder of your application with the customised file

## CSS Upgrading
To upgrade the CSS in this module, follow the [steps outlined in this repo](https://github.com/stadium-software/samples-upgrading)

# Optional Custom Development
1. Allowing users to save searches to the database (save generated search phase)
2. Providing a dropdown for users to apply previously saved searches (populate search field with previously saved phrase)
