# CurrentTime

## This is a repo for connecting to the Unit4 API
This repo lets you [GET], [POST], [PUT] and [DELETE]

## Routes:

```/get/<path>```

```/chained/<path>```

```/chained/<path>/<resource_path>```

```/chained/<path>/<resource_path>/<sub_resource_path>'```

```/post/<path>```

```/post/<path>/<resource_path>```

The microservice supports the use of since in Sesam. This is implemented in the ```/get/<path>``` route.

On the route ```/post/``` entities can be deleted and/or updated. Therefore use these routes with caution. Details on expected payloads for this functionality can be found in the above link to the documentation.

- To flag an entity for deletion in the CurrentTime API make sure you define a property with a key ```deleted``` and set its value to True.

- Additionally, make sure you provide the property with a key ```id``` and a respective value when needed on the ```/post/```routes.

- As of now, the ```/post/``` route does not support updating or deleting of a ```<sub_resource_path>```

## How to:

*Run program in development*

This repo uses the file ```package.json``` and [yarn](https://yarnpkg.com/lang/en/) to run the required commands.

1. Make sure you have installed yarn.
2. Creata a file called ```helpers.json``` and set current_url, current_user and current_password in the following format:
```
{
    "current_url": "some base_url",
    "current_user": "some username",
    "current_password": "some password"
}
```
3. run:
    ```
        yarn install
    ```
4. execute to run the script:
    ```
        yarn swagger
    ```

## Example payload for the /post paths route:

```

  [{ "payload": [{
      "employeeId": 1
      },{
        "employeeId": 2
    }]
  }]

```

### Config in Sesam

#### System example :

1. Name the system ```currenttime```

2. Config :

```
{
  "_id": "currenttime",
  "type": "system:microservice",
  "docker": {
    "image": "<docker username>/currenttime:<semantic_versioning>",
    "memory": 512,
    "port": 5000,
    "environment": {
      "current_user": "$ENV(<username for Unit4 account>)",
      "current_password": "$SECRET(<password for Unit4 account>)",
      "current_url": "$ENV(<your base_url>)"
    }
  },
  "verify_ssl": true
}
```

#### Pipe examples :

1. Name the pipe ```employees-currenttime```

2. Config :

```
{
  "_id": "employees-currenttime",
  "type": "pipe",
  "source": {
    "type": "json",
    "system": "currenttime",
    "is_since_comparable": true,
    "supports_since": true,
    "url": "/get/<path>"
  },
  "transform": {
    "type": "dtl",
    "rules": {
      "default": [
        ["copy", "*"],
        ["add", "_id",
          ["string", "_S.EmployeeId"]
        ]
      ]
    }
  },
  "pump": {
    "cron_expression": "0/10 * * * *",
    "rescan_cron_expression": "0 3 * * ?"
  }
}
```

1. Name the pipe ```employees-employeedetail```

2. Config :

```
{
  "_id": "employees-employeedetail",
  "type": "pipe",
  "source": {
    "type": "dataset",
    "dataset": "employees-currenttime"
  },
  "transform": {
    "type": "chained",
    "transforms": [{
      "type": "dtl",
      "rules": {
        "default": [
          ["filter",
            ["eq", "_S._deleted", false]
          ],
          ["add", "::payload",
            ["list",
              ["dict", "employeeId", "_S.EmployeeId"]
            ]
          ]
        ]
      }
    }, {
      "type": "http",
      "system": "currenttime",
      "batch_size": 1,
      "url": "/chained/employees/EmployeeDetail"
    }, {
      "type": "dtl",
      "rules": {
        "default": [
          ["copy", "*"],
          ["add", "_id",
            ["string", "_S.EmployeeDetailId"]
          ]
        ]
      }
    }]
  }
}

```

1. Name the pipe ```employees-employeedetail-outbound```

2. Config :

```
{
  "_id": "employees-employeedetail-outbound",
  "type": "pipe",
  "source": {
    "type": "dataset",
    "dataset": "employees-employeedetail"
  },
  "sink": {
    "type": "json",
    "system": "currenttime",
    "url": "/post/<path>" or "/post/<path>/<resource_path>"
  },
  "transform": {
    "type": "dtl",
    "rules": {
      "default": [
        ["copy", "*"],
        ["filter",
          ["eq", "_S._deleted", false]
        ]
      ]
    }
  }
}
```
