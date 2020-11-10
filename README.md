# amazon_lambda_node_example

# Common

- IAM 이용해서 permission 만들고 적용하기
- Cognito 생성시, javascript일 경우 clientId에서 `generate secretKey` 를 uncheck 해야한다.
- Body Mapping Template에 `When there are no templates defined (recommended)` check하기

# Post Request

### JSON schema

- CompareData

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "CompareData",
  "type": "object",
  "properties": {
    "age": {"type": "integer"},
    "height": {"type": "integer"},
    "income": {"type": "integer"}
  },
  "required": ["age", "height", "income"]
}
```

- CompareDataArray

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "CompareData",
  "type": "array",
  "items": {
      "type": "object",
        "properties": {
            "age": {"type": "integer"},
            "height": {"type": "integer"},
            "income": {"type": "integer"}
        }
  },
  "required": ["age", "height", "income"]
}
```

### Method Request

- Authorization
- Request Validator
- Request Body

### Integration Request

- Lambda Function

### Integration Request - Body Mapping Template

- Select 'When there are no templates defined'

```json
#set($inputRoot = $input.path('$'))
{
    "age": "$inputRoot.age",
    "height": "$inputRoot.height",
    "income": "$inputRoot.income",
    "userId": "$context.authorizer.claims.sub"
}
```

### Lambda function

```jsx
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({
    region: 'us-east-2', 
    apiVersion: '2012-08-10'
});

exports.handler = (event, context, callback) => {
    console.log(event);
    const params = {
        Item: {
            "UserId": {
                S: event.userId
            },
            "Age": {
                N: event.age
            },
            "Height": {
                N: event.height
            },
            "Income": {
                N: event.income
            }
        },
        TableName: "compare-yourself" 
    };
    dynamodb.putItem(params, function(err, data) {
        if(err) {
            console.log(err);
            callback();
        } else {
            console.log(data);
            callback(null, data);
        }
    });
};
```

# Delete Request

### Method Request

- Authorization

### Body Mapping Template, Integration Request

```json
{
    "userId": "$context.authorizer.claims.sub"
}
```

### Amazon Lambda

```jsx
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-2', apiVersion: '2012-08-10'});

exports.handler = (event, context, callback) => {
    const params = {
        Key: {
            "UserId": {
                S: event.userId
            }
        },
        TableName: "compare-yourself"
    };
    dynamodb.deleteItem(params, function (err, data) {
        if (err) {
            console.log(err);
            callback(err);     
        } else {
            console.log(data);
            callback(null, data);
        }
    });
};
```

# Get with `parameter`

- parameter

### Body Mapping Template - Request Integration

```json
{
    "type": "$input.params('type')",
    "accessToken": "$input.params('accessToken')"
}
```

### Body Mapping Template - Response Integration

```json
#set($inputRoot = $input.path('$'))
[
##TODO: Update this foreach loop to reference array from input json
#foreach($elem in $inputRoot)
 {
  "age" : $elem.age,
  "height" : $elem.height,
  "income" : $elem.income
} 
#if($foreach.hasNext),#end
#end
]
```

### Lambda function

```jsx
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-2', apiVersion: '2012-08-10'});
const cisp = new AWS.CognitoIdentityServiceProvider({apiVersion: '2016-04-18'});

exports.handler = (event, context, callback) => {
    const accessToken = event.accessToken;
    
    const type = event.type;
    if (type == 'all') {
        const params = {
            TableName: 'compare-yourself'
        };
        dynamodb.scan(params, function(err, data) {
            if (err) {
                console.log(err);
                callback(err);
            } else {
                console.log(data);
                const items = data.Items.map(
                    (dataField) => {
                        return {age: +dataField.Age.N, height: +dataField.Height.N, income: +dataField.Income.N};
                    }    
                );
                callback(null, items);
            }
        });
    } else if (type == 'single') {
        const cispParams = {
            "AccessToken": accessToken
        };
        
        cisp.getUser(cispParams, (err, result) => {
            if (err) {
                console.log(err);
                callback(err);
            } else {
                console.log(result);
                const userId = result.UserAttributes[0].Value;
                
                const params = {
                    Key: {
                        "UserId": {
                            S: userId
                        }
                    },
                    TableName: "compare-yourself"
                };
                dynamodb.getItem(params, function(err, data) {
                    if (err) {
                        console.log(err);
                        callback(err);
                    } else {
                        console.log(data);
                        callback(null, [{age: +data.Item.Age.N, height: +data.Item.Height.N, income: +data.Item.Income.N}]);    
                    }
                });
            }
        });
    } else {
        callback('Something went wrong!');   
    }
};
```