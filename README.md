# DataJoint SciViz

- DataJoint SciViz is a generic scientific visualization framework for building dashboards of DataJoint pipelines.

## References

- [DataJoint Documentation](https://datajoint.com/docs)
- [DataJoint SciViz Documentation](https://datajoint.com/docs/core/sci-viz)
- [Pharus (a DataJoint REST API backend)](https://github.com/datajoint/pharus)

## Installation

If you have not done so already, please install the following dependencies:

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Prerequisites

Before running the application through docker, make sure to run the command:

```bash
python frontend_gen.py
```

This will compile the necessary typescript files for the frontend and the necessary python files for the backend.
you can also run this at any time during a dev deployment to hot-load the frontend, however this is not always guaranteed to work as some changes may require the entire react server to restart. Anything related to the back end api is not hot-loadable but for example the component locations and sizes can be modified and hot-loaded.

Also see the _.env_ section to set up your environment variables.

## Running the application

To start the application in dev mode, use the command:

```bash
PY_VER=3.8 IMAGE=djbase DISTRO=alpine PHARUS_VERSION=$(cat pharus/pharus/version.py | tail -1 | awk -F\' '{print $2}') HOST_UID=$(id -u) docker-compose up --build
```

To run the application in Production mode, use the command:

```bash
docker-compose -f docker-compose-deploy.yaml up --build
```

To stop the application, use the same command as before but with `down` in place of `up --build`

## Dynamic spec sheet

Sci-Viz is used to build visualization dashboards, this is done through a single spec sheet. An example spec sheet is included named `example_visualization_spec.yaml`

<details>
<summary>Click to expand details</summary>

Important notes about restrictions in the spec sheet:

- Page names under pages must have a unique name without spaces
- Page routes must be unique
- Grid names under grids must be unique without spaces
- Component names under components must be unique **but can have spaces**
- The routes of individual components must be unique
- Routes must start with a `/`
- Every query needs a restriction, below is the default one.
  - ```python
        def restriction(**kwargs):
            return dict(**kwargs)
    ```
- Overlapping components at the same (x, y) does not work, the grid system will not allow overlapping components it will wrap them horizontally if there is enough space or bump them down to the next row.

If the website does not work after running the frontend generation script check this list to make sure that spec sheet is constructed properly, in the future we may include a script that lints the spec sheet for you. see issue [#20](https://github.com/datajoint/sci-viz/issues/20)

</details>

## Grids

Sci-Viz produces custom visualizations by putting `grids` on `pages` and then filling them with visualization `components`. Currently there are two types of grids **Fixed** and **Dynamic**

<details>
<summary>Click to expand details</summary>

### Fixed mode grid

A fixed mode grid requires all components to explicitly give their position and size on the grid.

A fixed `grid` takes 4 arguments:

- `type:` indicates the type of grid, in this case `type: fixed`
- `columns:` the number of columns that the grid will have
- `row_height:` the height of each row in pixels
- `components:` a yaml dictionary of components to be spawned in the grid

### Dynamic grid mode

A dynamic grid takes a datajoint query and then uses each record and applies that record as a restriction to a template of components. It then spawns a single or group of components for each record of that parent query but the components query is restricted by the entire record that has been passed in from the parent query.

An example of this would be as follows:

- You have one table that represents all identifying data of a subject, lets use Mouse as an example for the subject and the table name
- You also have a table that contains a single plot per Mouse primary key, lets call this table MousePlots
- You have no idea how many plots are in MousePlots but you want to display a live view of all of them
- What you can do is create a dynamic grid with the parent query being for the Mouse table and a plot component with a query for the MousePlot table. This will produce all of the plots that are available without knowing how many there are in the database.

A dynamic `grid` takes 7 arguments:

- `type:` indicates the type of grid, in this case `type: fixed`
- `columns:` the number of columns that the grid will have
- `row_height:` the height of each row in pixels
- `restriction:` a restriction for the datajoint query
- `dj_query:` the parent datajoint query that will provide the restriction records
- `route:` backend api route for the parent query
- `component_templates:` a yaml dictionary of components that serve as a template

Additionally any components in the dynamic grid do not need `x`, `y` , `height`, and `width` fields.

Currently only the `metadata` and `plot` components are supported in dynamic mode.

</details>

## Components

<details>
<summary>Click to expand details</summary>

All components need minimally these fields:

- `type:` indicates the type of component you are trying to generate
- `x:` x position on the grid starting at 0
- `y:` y position on the grid starting at 0
- `height:` the amount of grid squares tall a component can be, minimum 1
- `width:` the amount of grid square wide a compoentnt can be, minimum 1

### Table component

`type:` table

The Table component takes a few additional fields:

- `route:` the backend route for the rest api query, must start with a `/`
- `restriction:` the restriction for the datajoint query
- `dj_query:` the datajoint query for for the table data

If setup correctly the component will render the result of the query in a table that supports paging, sorting, and filtering.

#### Adding color to your tables using projections

```python
def dj_query(vms):
    TableA, TableB = (vms['test_group1_simple'].TableA, vms['test_group1_simple'].TableB)
    return ((TableA * TableB).proj(...,
                                   _sciviz_font='IF(a_name = "Raphael", "rgb(255, 0, 0)", NULL)',
                                   _sciviz_background='IF(a_name = "Raphael", "rgba(50, 255, 0, 0.16)", NULL)',)
                                  ), dict(order_by='b_number')
```

This is an example of a table query that has a projection that applys a text color as well as a background color.
It does so through the use of 2 protected column names:

- `_sciviz_font` for the font color
- `_sciviz_background` for the background color
  these two fields will accept any color format that css does.

In the example we do a join of two tables and then do a projection where we create 2 new columns with the protected names and if a condition is met we set their field to a css-compatable color else we have it be `NULL`. In the example above we use rgb when we do not need transparency and rgba when we do.
[here is a good tool for picking css colors.](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Colors/Color_picker_tool)

### Markdown component

`type:` markdown

The markdown component takes one additional field `text: |`
underneath the `|` operator you can place any markdown text block that you want.

### Plot component from stored Plotly JSON

`type:` plot:plotly:stored_json

The plot component takes 3 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`
- `restriction:` the restriction for the datajoint query
- `dj_query:` the datajoint query for for the table data.

The plot component also takes one optional argument:

- `channels:` (string Arr) channels to listen to for additional restrictions from other components (slider, dropdown, ect.)

Additionally for the plot to render properly the result of your query must be a single entry with one element that is a plotly JSON.
An easy way to do this is to set the `fetch_args=[]` in your `dj_query` to be only the column that contains a plotly JSON and additionaly set your restriction to be the index of the plot you are looking for

### Metadata component

`type:` metadata

The Metadata component takes 3 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`
- `restriction:` the restriction for the datajoint query
- `dj_query:` the datajoint query for for the table data.

Additionally the metadata component only takes a single row from a table as its input so the `dj_query` and `restriction` need to be properly set to produce a single record. This component is not very useful by itself but when combined with other components as part of a template in a `Dynamic grid` it can provide useful information on what the other components are showing.

### Image component

`type:` file:image:attach

the Image component takes 3 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`
- `restriction:` the restriction for the datajoint query
- `dj_query:` the datajoint query for for the table data.

Additionally the image that you want to display needs to be stored as a datajoint [attach](https://docs.datajoint.org/python/definition/06.5-External-Data.html?highlight=attach) attribute type and your query should produce only one record with one column which is the column where the image is stored.

### Form component

`type:` form

The Form component takes 2 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`
- `tables:` the list of tables in "`schema.table`" format to insert into
  - Table names can be templated, either fully or partially, using the `'{keyword}'` format. This keyword can then be assigned a value by an emitter component as a query parameter.

The Form can also take 2 optional argument:

- `map:` a mapping to change the displayed names of the fields in the form

  A map takes a _list_ of 3 arguments:

  - `type:` attribute | table
  - `input:` the new name of the field
  - `destination:` the field to be renamed

  A map entry with a `table` type can also take 1 optional argument:

  - `map:` a nested mapping of the same structure to change the displayed names of the table's primary key attributes

- `channels:` an array of channels to listen to for templated table name values.

### Slider component

The slider is a component that takes a datajoint query and creates a slider based off the payload that the query returns. It turns each record into an index on the slider and also emits the currently selected record on its channel as a restriction to other components.

`type:` slider

the Slider component takes 3 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`.
- `restriction:` the restriction for the datajoint query.
- `dj_query:` the datajoint query for for the table data.
- `channel:` the name of the channel that the slider outputs its restriction on.

The Slider also takes one optional argument:

- `channels:` the array of channels to listen to for restricting its own query.

### Radiobutton/dropdown-static component

Similar to the Slider, the radiobutton and dropdown-static components are components that supply a selected restriction on a channel to a component that can accept them.

`type:` radiobuttons | dropdown-static

the Radiobutton/dropdown-static component takes 2 additional arguments:

- `channel:` the name of the channel that the Radiobutton/dropdown-static outputs its restriction on.
- `content:` dictionary of key value pairs, the key is what text is shown to the user while the value is the actual restriction. example:

```
content:
  mouse 0: 'mouse_id=0'
  mouse 1: 'mouse_id=1'
  mouse 2: 'mouse_id=2'
```

### dropdown-query component

The dropdown-query component is the same as the slider component except it only expects a result with one column.

`type:` dropdown-query

the dropdown-query component takes 3 additional arguments:

- `route:` the backend route for the rest api query, must start with a `/`.
- `restriction:` the restriction for the datajoint query.
- `dj_query:` the datajoint query for for the table data.
- `channel:` the name of the channel that the slider outputs its restriction on.

</details>

## Developer instructions

There are a couple issues to address if you are collaborating on this project

- devs will have have to point the submodule to their own fork of pharus if they need to edit pharus to support new features for sci-viz.
- That change to pharus would need to be pr'd and then merged into pharus before we can pr and merge their change to sci-viz as we probably dont want unreviewed code linked to sci-viz nor do we want the submodule pointing to their fork of pharus.

### .env

for running frontend_gen.py you need this variable in your .env file

```bash
DJSCIVIZ_SPEC_PATH=test/test_spec.yaml
```
