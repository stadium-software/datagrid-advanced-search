# DataGrid Client Side Filters

Building advanced client-side search forms for DataGrids

https://github.com/user-attachments/assets/e62cbf4d-bbd1-481e-a7f3-11e7dfbf9bf3

## Content
- [DataGrid Client Side Filters](#datagrid-client-side-filters)
  - [Content](#content)
  - [Version](#version)
    - [Change Log](#change-log)
  - [Application Setup](#application-setup)
  - [Database, Connector and DataGrid](#database-connector-and-datagrid)
  - [Global Script Setup](#global-script-setup)
  - [FilterConfig Type Setup (Required)](#filterconfig-type-setup-required)
  - [SelectedFilters Type Setup (Optional)](#selectedfilters-type-setup-optional)
  - [Page Setup](#page-setup)
  - [Page.Load Setup](#pageload-setup)
  - [Display Modes](#display-modes)
    - [Default](#default)
    - [Collapsed](#collapsed)
    - [Integrated](#integrated)
  - [Callback Script Setup](#callback-script-setup)
  - [Programatically Applying Filters](#programatically-applying-filters)
  - [Applying the CSS](#applying-the-css)
  - [Customising CSS](#customising-css)
  - [CSS Upgrading](#css-upgrading)
  - [Known Issues](#known-issues)

## Version
Current version 3.2

### Change Log
2.0 Complete rewrite of the feature. Simplified setup by generating all form elements in JS script. Added [display modes](#display-modes) (standard, collapsed and integrated)

2.1 Fixed selectable column bug

2.2 Changed Save/Apply button text

2.3 Fixed url parsing bug

2.4 Switched column parameter from [heading property to name property](#pageload-setup); fixed number and date input display bug; general JS cleanup

2.5 Fixed "control in template" bug; fixed "invisible column" filter bug; fixed non-existent column number filter bug; fixed integrated display bug (CSS)

2.6 Added "Equals" condition for number and date types

2.6.1 Added CSS variable to reverse the order of the "Apply" and "Clear" buttons (CSS only)

2.7 Fixed "Selectable Data" bug

2.8 Fixed "From-To" top margin display bug (CSS only)

2.9 Bug fixes and enhancements:
1. Fixed "Integrated" collapsed display bug
2. Fixed Number & Date "not to value provided" search error
3. Added Date "format" option
4. Added Date "display" option

3.0 Bug fixes and enhancements
1. Enable enter key press to search when focus is in a value field
2. Optional 'operator' list property in 'FilterConfig' type to limit field operators for text, number & date filters
3. Added "From-To" operator for date fields
4. Fixed "picker" filter date not showing bug
5. Fixed "Between" dates bug
6. Made "From-To" the default operator for dates and numbers

3.0.1 Bug fix 'equals' operator

3.2 Enhancements
1. Added 'Callback' parameter; when provided this script returns applied filters to the callback script
2. Added 'SelectedFilters' parameter to apply filters programatically

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
   7. Callback
   8. SelectedFilters
3. Drag a Javascript action into the script and paste the Javascript below unaltered into the action
```javascript
/* Stadium Script v3.2 https://github.com/stadium-software/datagrid-advanced-search */
let scope = this;
let filterClassName = "." + ~.Parameters.Input.FilterContainerClass;
let classInput = ~.Parameters.Input.DataGridClass;
if (typeof classInput == "undefined") {
    console.error("The DataGridClass parameter is required");
    return false;
}
let dgClassName = "." + classInput;
let filterConfig = ~.Parameters.Input.FilterConfig;
let callback = ~.Parameters.Input.Callback;
let selectedFilters = ~.Parameters.Input.SelectedFilters || [];
let displayMode = ~.Parameters.Input.DisplayMode;
if (displayMode) displayMode = displayMode.toLowerCase();
const insert = (arr, index, newItem) => [...arr.slice(0, index), newItem, ...arr.slice(index)];
let numberSelectChange = (e) => {
    let target = e.target;
    let toEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-to-number");
    let fromEl = target.closest("div").nextElementSibling.querySelector(".filtergrid-from-number");
    if (target.value != "Between" && target.value != "From-To") {
        toEl.classList.add("visually-hidden");
        toEl.value = "";
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
    let targetVal = target.value.toLowerCase();
    if (targetVal == "greater than" || targetVal == "smaller than" || targetVal == "equals") {
        toEl.classList.add("visually-hidden");
        toEl.value = "";
        fromEl.setAttribute("placeholder", "Date");
    } else { 
        toEl.classList.remove("visually-hidden");
        fromEl.setAttribute("placeholder", "From date");
    }
};
let tries = 0;
let wait = async (milliseconds) => new Promise((resolve) => setTimeout(resolve, milliseconds));
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    console.error("The class '" + dgClassName + "' is not assigned to any DataGrid");
    return false;
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
}
dg = dg[0];
dg.classList.add("stadium-filtered-datagrid");
let getObjectName = (obj) => {
    let objname = obj.id.replace("-container","");
    do {
        let arrNameParts = objname.split(/_(.*)/s);
        objname = arrNameParts[1];
    } while ((objname.match(/_/g) || []).length > 0 && !scope[`${objname}Classes`]);
    return objname;
};
let datagridname = getObjectName(dg);
let dataGridColumns = getColumnDefinition();
let filterContainer = document.querySelectorAll(filterClassName);
if (filterContainer.length == 0) {
    console.error("The container for the filter was not found. Drag a container control into the page and assign the class '" + filterClassName + "' to it.");
    return false;
} else if (dg.length > 1) {
    console.error("The class '" + filterClassName + "' is assigned to multiple controls. Assign a unique classname to the filter container");
    return false;
}
filterContainer = filterContainer[0];
let filterInnerContainer = filterContainer.querySelector(".stadium-filter-inner-container"),
    stadiumFilters  = filterContainer.querySelector(".stadium-filters");
if (!filterInnerContainer){
    filterContainer.classList.add("stadium-filter-container");
    filterInnerContainer = document.createElement("div");
    filterInnerContainer.classList.add("stadium-filter-inner-container");
    filterContainer.appendChild(filterInnerContainer);
    stadiumFilters = document.createElement("div");
    stadiumFilters.classList.add("stadium-filters");
    stadiumFilters.setAttribute("forDG",dg.id);
    filterInnerContainer.appendChild(stadiumFilters);
    initFilterForm();
}
if (selectedFilters.length > 0) {
    setSelectedFilters();
}

function initFilterForm() {
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
            if (!columnDef) {
                console.error("Column '" + column + "' was not found.");
                continue;
            }
            column = columnDef.name;
        }
        if (columnDef.visible == false) {
            console.error("Column '" + column + "' is not visible. Only visible columns can be used. ");
            continue;
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
        let format = filterConfig[i].format;
        let operators = filterConfig[i].operators || [];
        operators = operators.map(v => v.toLowerCase());

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
            let options = ["Contains", "Does not contain", "Equals", "Does not equal"];
            for (let s = 0; s < options.length; s++) {
                let opt = options[s];
                if (operators.includes(opt.toLowerCase()) || operators.length == 0) {
                    let el = document.createElement("option");
                    el.textContent = opt;
                    el.value = opt;
                    if (operators.length == 1) select.setAttribute("readonly", "readonly");
                    select.classList.add("filter-operator");
                    select.appendChild(el);
                }
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            input = document.createElement("input");
            input.classList.add("form-control", "text-box-input", "filtergrid-text-value");
            input.setAttribute("placeholder", "Text");
            input.addEventListener("keypress", applyOnKeypress);
            valueField.classList.add("control-container","text-box-container");
        }
        if (type == "number") {
            select = document.createElement("select");
            let options = ["From-To", "Between", "Equals", "Greater than", "Smaller than"];
            for(let s = 0; s < options.length; s++) {
                let opt = options[s];
                if (operators.includes(opt.toLowerCase()) || operators.length == 0) {
                    let el = document.createElement("option");
                    el.textContent = opt;
                    el.value = opt;
                    if (operators.length == 1) select.setAttribute("readonly", "readonly");
                    select.classList.add("filter-operator");
                    select.appendChild(el);
                }
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            let numInput1 = document.createElement("input");
            numInput1.classList.add("form-control", "text-box-input", "filtergrid-from-number");
            numInput1.setAttribute("placeholder", "From value");
            numInput1.addEventListener("keypress", applyOnKeypress);
            let numInput2 = document.createElement("input");
            numInput2.classList.add("form-control", "text-box-input", "filtergrid-to-number");
            numInput2.setAttribute("placeholder", "To value");
            numInput2.addEventListener("keypress", applyOnKeypress);
            input = document.createElement("div");
            input.classList.add("number-values");
            select.addEventListener("change", numberSelectChange);
            input.appendChild(numInput1);
            input.appendChild(numInput2);
        }
        if (type == "date") {
            if (!format) format = 'YYYY/MM/DD';
            select = document.createElement("select");
            let options = ["From-To", "Between", "Equals", "Greater than", "Smaller than"];
            for(let s = 0; s < options.length; s++) {
                let opt = options[s];
                if (operators.includes(opt.toLowerCase()) || operators.length == 0) {
                    let el = document.createElement("option");
                    el.textContent = opt;
                    el.value = opt;
                    if (operators.length == 1) select.setAttribute("readonly", "readonly");
                    select.classList.add("filter-operator");
                    select.appendChild(el);
                }
            }
            select.classList.add("form-control");
            operator.classList.add("control-container", "drop-down-container");
            let dtInput1 = document.createElement("input");
            dtInput1.classList.add("form-control", "text-box-input", "filtergrid-from-date");
            dtInput1.setAttribute("placeholder", "From date");
            dtInput1.setAttribute("format", format);
            dtInput1.addEventListener("keypress", applyOnKeypress);
            let dtInput2 = document.createElement("input");
            dtInput2.classList.add("form-control", "text-box-input", "filtergrid-to-date");
            dtInput2.setAttribute("placeholder", "To date");
            dtInput2.addEventListener("keypress", applyOnKeypress);
            if (display == "picker") {
                dtInput1.type = "date";
                dtInput2.type = "date";
            }
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
    saveButton.classList.add("btn", "btn-lg", "btn-default", "apply-button");
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

async function filterDataGrid(){
    let searchPhrase = [];
    let arrReturn = [];
    let operatorEls = stadiumFilters.querySelectorAll("[foperator]");
    for (let i = 0; i < operatorEls.length; i++) {
        let ftype = operatorEls[i].getAttribute("ftype");
        let colNo = operatorEls[i].getAttribute("cno");
        let fdisplay = operatorEls[i].getAttribute("fdisplay");
        let foperator = operatorEls[i].getAttribute("foperator");
        let colText = dataGridColumns[colNo - 1].headerText;
        let fvalueEl = operatorEls[i].nextElementSibling;
        let output, heading = "";
        if (colText) heading = colText.replaceAll(" ", "\\ ");
        if (ftype == "text") {
            let txtoperator = operatorEls[i].querySelector("select").value;
            let txtvalue = fvalueEl.querySelector("input").value;
            if (txtoperator == "Contains" && txtvalue) {
                output = heading + ':"' + txtvalue + '"';
            } else if (txtoperator.toLowerCase() == "does not contain" && txtvalue) {
                output = heading + ':(NOT "' + txtvalue + '")';
            } else if (txtoperator.toLowerCase() == "equals" && txtvalue) {
                output = heading + ':["' + txtvalue + '" TO "' + txtvalue + '"]';
            } else if (txtoperator.toLowerCase() == "does not equal" && txtvalue) {
                output = heading + ':(NOT ["' + txtvalue + '" TO "' + txtvalue + '"])';
            }
            if (txtvalue) arrReturn.push({"column": foperator, "selectedoperator":txtoperator, selectedvalues: [txtvalue]});
        }
        if (ftype == "number") {
            let numoperator = operatorEls[i].querySelector("select").value;
            let numvaluefromEl = fvalueEl.querySelector(".filtergrid-from-number");
            let numvaluetoEl = fvalueEl.querySelector(".filtergrid-to-number");
            let numvaluefrom = numvaluefromEl.value;
            let numvalueto = numvaluetoEl.value;
            if (numvaluefrom) {
                if (!numvaluefrom) numvaluefrom = -9007199254740991;
                if (!numvalueto) numvalueto = 9007199254740991;
                numvaluefromEl.value = numvaluefrom;
                numvaluetoEl.value = numvalueto;
                if (numoperator.toLowerCase() == "between") {
                    output = heading + ':{' + numvaluefrom + ' TO ' + numvalueto + '}';
                } else if (numoperator.toLowerCase() == "from-to") {
                    output = heading + ':[' + numvaluefrom + ' TO ' + numvalueto + ']';
                } else if (numoperator.toLowerCase() == "equals") {
                    output = heading + ':[' + numvaluefrom + ' TO ' + numvaluefrom + ']';
                } else if (numoperator.toLowerCase() == "greater than") {
                    output = heading + ':{' + numvaluefrom + ' TO 9007199254740991}';
                } else if (numoperator.toLowerCase() == "smaller than") {
                    output = heading + ':{-9007199254740991 TO ' + numvaluefrom + '}';
                }
                arrReturn.push({"column": foperator, "selectedoperator":numoperator, selectedvalues: [numvaluefrom, numvalueto]});
            }
        }
        if (ftype == "date") {
            let dtoperator = operatorEls[i].querySelector("select").value;
            let fromEl = fvalueEl.querySelector(".filtergrid-from-date");
            let toEl = fvalueEl.querySelector(".filtergrid-to-date");
            let format = fromEl.getAttribute("format");
            if (fromEl.value) {
                let dtvaluefrom = dayjs(fromEl.value).format(format);
                let dtvalueto = dayjs(toEl.value).format(format);
                if (dtvaluefrom == "Invalid Date") dtvaluefrom = dayjs('1000/01/01').format(format);
                if (dtvalueto == "Invalid Date") dtvalueto = dayjs('3000/01/01').format(format);
                if (fromEl.type == "date") {
                    fromEl.value = dayjs(dtvaluefrom).format('YYYY-MM-DD');
                    toEl.value = dayjs(dtvalueto).format('YYYY-MM-DD');
                } else {
                    fromEl.value = dtvaluefrom;
                    toEl.value = dtvalueto;
                }
                if (dtoperator.toLowerCase() == "between") {
                    output = heading + ':{' + dtvaluefrom + ' TO ' + dtvalueto + '}';
                } else if (dtoperator.toLowerCase() == "from-to") {
                    output = heading + ':[' + dtvaluefrom + ' TO ' + dtvalueto + ']';
                } else if (dtoperator.toLowerCase() == "equals") {
                    output = heading + ':[' + dtvaluefrom + ' TO ' + dayjs(dtvaluefrom).add(1, 'day').format(format) + ']';
                } else if (dtoperator.toLowerCase() == "greater than") {
                    output = heading + ':{' + dtvaluefrom + ' TO ' + dayjs('3000/01/01').format(format) + '}';
                } else if (dtoperator.toLowerCase() == "smaller than") {
                    output = heading + ':{' + dayjs('1000/01/01').format(format) + ' TO ' + dtvaluefrom + '}';
                }
                arrReturn.push({"column": foperator, "selectedoperator":dtoperator, selectedvalues: [dtvaluefrom, dtvalueto]});
            }
        }
        let selectedvals = [];
        if (ftype == "boolean" && fdisplay == "radio") {
            let multioperator = operatorEls[i].querySelectorAll("input[type='radio']");
            for (let s = 0; s < multioperator.length; s++) {
                if (multioperator[s].checked && multioperator[s].value != "Show all") {
                    output = heading + ':' + multioperator[s].value;
                    selectedvals.push(multioperator[s].value);
                }
            }
        } else if (ftype == "boolean") {
            let booloperator = operatorEls[i].querySelector("select").value;
            if (booloperator != "Show all" && booloperator != "") {
                output = heading + ':' + booloperator;
                selectedvals.push(booloperator);
            }
        }
        if (ftype == "enum" && fdisplay == "radio") {
            let multioperator = operatorEls[i].querySelectorAll("input[type='radio']");
            for (let s = 0; s < multioperator.length; s++) {
                if (multioperator[s].checked && multioperator[s].value != "Show all") {
                    output = heading + ':["' + multioperator[s].value + '" TO "' + multioperator[s].value + '"]';
                    selectedvals.push(multioperator[s].value);
                }
            }
        } else if (ftype == "enum") {
            let enumoperator = operatorEls[i].querySelector("select").value;
            if (enumoperator != "Show all" && enumoperator != "") {
                output = heading + ':["' + enumoperator + '" TO "' + enumoperator + '"]';
                selectedvals.push(enumoperator);
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
                    selectedvals.push(multioperator[s].value);
                }
            }
            output += ")";
        }
        if (selectedvals.length > 0) arrReturn.push({"column": foperator, selectedvalues: selectedvals});
        if (output && output != "()") searchPhrase.push(output);
    }
    filterContainer.classList.remove("expand");
    scope[`${datagridname}SearchTerm`] = searchPhrase.join(' AND ');
    if (callback) {
        scriptCaller(callback, arrReturn);
    }
}

function setSelectedFilters(){
    clearForm();
    for (let i=0;i < selectedFilters.length;i++) {
        let column = selectedFilters[i].column, 
            selectedoperator = selectedFilters[i].selectedoperator,
            selectedvalues = selectedFilters[i].selectedvalues,
            operator = filterContainer.querySelector("[foperator='" + column + "']"),
            value = filterContainer.querySelector("[fvalue='" + column + "']"),
            select, radioinputs, checkinputs, display, type;
        if (operator) {
            display = operator.getAttribute("fdisplay");
            type = operator.getAttribute("ftype");
            radioinputs = operator.querySelectorAll("input[type='radio']");
            checkinputs = operator.querySelectorAll("input[type='checkbox']");
            select = operator.querySelector("select");
        }
        if (type == "text") {
            if (selectedoperator && [...select.options].map(el => el.value).includes(selectedoperator.toString())) select.value = selectedoperator;
            if (selectedvalues && selectedvalues.length > 0) value.querySelector("input").value = selectedvalues[0];
        }
        if (type == "number") {
            if (selectedoperator && [...select.options].map(el => el.value).includes(selectedoperator.toString())) select.value = selectedoperator;
            if (selectedvalues && selectedvalues.length > 0) value.querySelectorAll("input")[0].value = selectedvalues[0];
            if (selectedvalues && selectedvalues.length > 0) value.querySelectorAll("input")[1].value = selectedvalues[1];
        }
        if (type == "date") {
            if (selectedoperator && [...select.options].map(el => el.value).includes(selectedoperator.toString())) select.value = selectedoperator;
            if (selectedvalues && selectedvalues.length > 0) value.querySelectorAll("input")[0].value = selectedvalues[0];
            if (selectedvalues && selectedvalues.length > 0) value.querySelectorAll("input")[1].value = selectedvalues[1];
        }
        if (type == "boolean") {
            if (display == "radio") {
                for (let s = 0; s < radioinputs.length; s++) {
                    if (selectedvalues && selectedvalues.length > 0 && selectedvalues[0] == radioinputs[s].value) radioinputs[s].checked = true;
                }
            } else {
                if (selectedvalues && selectedvalues.length > 0 && [...select.options].map(el => el.value).includes(selectedvalues[0].toString())) select.value = selectedvalues[0];
            }
        }
        if (type == "enum") {
            if (display == "radio") {
                for (let s = 0; s < radioinputs.length; s++) {
                    if (selectedvalues && selectedvalues.length > 0 && selectedvalues[0] == radioinputs[s].value) radioinputs[s].checked = true;
                }
            } else {
                if (selectedvalues && selectedvalues.length > 0 && [...select.options].map(el => el.value).includes(selectedvalues[0].toString())) select.value = selectedvalues[0];
            }
        }
        if (type == "multiselect") {
            for (let s = 0; s < checkinputs.length; s++) {
                if (selectedvalues && selectedvalues.length > 0 && selectedvalues.includes(checkinputs[s].value)) checkinputs[s].checked = true;
            }
        }
        if (select) select.dispatchEvent(new Event('change'));
    }
    filterDataGrid();
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
    let operators = stadiumFilters.querySelectorAll(".filter-operator");
    for (let i = 0; i < operators.length; i++) {
        operators[i].dispatchEvent(new Event('change'));
    }
    scope[`${datagridname}SearchTerm`] = null;
}
function applyOnKeypress(e){
    if(e.keyCode === 13){
        e.target.closest(".stadium-filters").querySelector(".apply-button").click();
    }
}
function setAttributes(el, attrs) {
  for(var key in attrs) {
    el.setAttribute(key, attrs[key]);
  }
}
function getColumnDefinition() {
    let cols = [];
    if (scope[`${datagridname}HasSelectableData`]) {
        cols.push({name:"RowSelector", headerText: "RowSelector"});
    }
    let colDefs = scope[`${datagridname}ColumnDefinitions`];
    for (let i=0;i<colDefs.length;i++) {
        cols.push(colDefs[i]);
    }
    return cols;
}
function isNumber(str) {
    if (typeof str == "number") return true;
    return !isNaN(str) && !isNaN(parseFloat(str));
}
function getElementFromObjects(haystack, needle, column) {
    return haystack.find(obj => {return obj[column] == needle;});
}
async function scriptCaller(script, data) {
    tries++;
    if (tries > 20) {
    	return false;
    } else {
        try {
            await scope[script](data);
            return true;
        } catch (error) {
            wait(100).then(() => scriptCaller(script, data));
        }
    }
}
```

## FilterConfig Type Setup (Required)
1. Add a type called "FilterConfig" to the types collection in the Stadium Application Explorer
2. Add the following properties to the type
   1. type (Any)
   2. name (Any)
   3. column (Any)
   4. display (Any)
   5. data (List)
      1. Item (Any)
   6. format (Any)
   7. operators (List)
      1. Item (Any)

![Filters COnfigType Setup](images/TypeSetup.png)

## SelectedFilters Type Setup (Optional)
1. If you wish to apply filters programatically, it may be useful to create a second type called "SelectedFilters"
2. Add the following properties to the type
   1. column (Any)
   2. selectedoperator (Any)
   3. selectedvalues (List)
      1. Item (Any)

![Applied Filters Type](images/AppliedFiltersType.png)

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
   4. *display*
      1. For *boolean* and *enum* types: These are shown as dropdowns by default, but passing the value "radio" in this property will cause them to be shown as radio button lists instead
      2. For *date* type (optional; default is false): Add "picker" to display a browser-provided HTML Date Picker in the date input fields
   5. *data*: filters of type *enum* and *multiselect* require a list of data users can select from
   6. *format* (*date* type only; default 'YYYY/MM/DD'): The format in which date filter values are passed to the DataGrid search box. The module uses [DayJS Formats](https://day.js.org/docs/en/display/format)
   7. *operators* (list; optional; only for types 'text', 'date' and 'number'): A list of operators to show in the operators dropdown. Use when you want to show only a subset of the options below. Allowed operators
      1. text: "Contains", "Does Not Contain", "Equals", "Does Not Equal"
      2. number: "Between", "From-To", "Equals", "Greater than", "Smaller than"
      3. date: "Between", "From-To", "Equals", "Greater than", "Smaller than"

Fields Definition Example
```json
= [{
	"type": "text",
	"name": "First Name",
	"column": "FirstName",
    "operators": ["Contains", "Does Not Contain"]
},{
    "type": "text",
    "name": "Last Name",
    "column": 3
},{
	"type": "date",
	"name": "Start Date",
	"column": "StartDate",
    "display": "picker",
    "operators": ["Between", "From-To", "Equals"]
},{
	"type": "date",
	"name": "End Date",
	"column": "EndDate",
    "format": "YYYY-MM-DD",
    "operators": ["Greater than", "Smaller than"]
},{
	"type": "number",
	"name": "Number Of Pets",
	"column": "NoOfPets",
    "operators": ["Between", "From-To", "Equals"]
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
   7. Callback (optional): The script will attempt to return the applied filters to a callback script (see [Callback Script Setup](#callback-script-setup))
   8. SelectedFilters (optional): A *List* of type "SelectedFilters" (see [SelectedFilters Type Setup (Optional)](#selectedfilters-type-setup-optional))

![Script Parameters Example](images/ScriptParametersExample.png)

## Display Modes
There are three display modes:
1. default
2. collapsed
3. integrated

### Default
1. Leaving the "DisplayMode" parameter empty means the filter is showing as a block wherever the *Container* control is placed on the page

![Standard Display](images/Standard.gif)

### Collapsed
2. Adding the word "collapsed" in the parameter causes the filter to be expandable by the user. When expanded, the filter will displace any controls below it on the page

![Collapsed Display](images/Collapsed.gif)

### Integrated
3. Adding the word "integrated" in the parameter causes the filter to be expandable and shown as an icon next to the search bar. When it is opened, the filter will overlay other controls onthe page

![Integrated Display](images/Integrated.gif)

## Callback Script Setup
In order to get back a list of filters that were applied to a DataGrid from the script:
1. Add a page script (choose any name, e.g. FiltersCallback)
2. Add an input parameter to the script (e.g. AppliedFilters)
3. Add the script name (e.g. FiltersCallback) to the "Callback" parameter of the "DataGridFilter" script
4. The input parameter of your page script will contain a list of type "SelectedFilters" that you can save and assign to the "SelectedFilters" parameter later

## Programatically Applying Filters
In order to apply filters from scripts or event handlers (for example defaults or saved sets) you need to
1. Create a *List* or type "SelectedFilters"
2. Populate that list with an object for each filter you wish to apply
   1. column: The name of the DataGrid column
   2. selectedoperator: The value for the operator (only needed for text, number and date types)
   3. selectedvalues: A List of type "Any" containing the values to filter by (first value will be 'From' and second value 'To' where applicable). 

## Applying the CSS
The CSS below is required for the correct functioning of the module. Some elements can be [customised](#customising-css) using a variables CSS file. 

**Stadium 6.6 or higher**
1. Create a folder called "CSS" inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-custom-filters-variables.css*](datagrid-custom-filters-variables.css) and [*datagrid-custom-filters.css*](datagrid-custom-filters.css) into that folder
3. Paste the link tags below into the "head" property of your application
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

## Known Issues
1. Using an underscore (_) in the page or template name can cause the script to break
