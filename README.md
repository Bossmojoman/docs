# Triage API Functions

## CD/CD - Azure Pipelines Information

[![Build Status](https://dev.azure.com/Healthwise-Platform/Platform%20Team/_apis/build/status/Triage%20API?branchName=master)](https://dev.azure.com/Healthwise-Platform/Platform%20Team/_build/latest?definitionId=25&branchName=master)

- [Triage API Functions](#triage-api-functions)
  - [CD/CD - Azure Pipelines Information](#cdcd---azure-pipelines-information)
    - [Notes](#notes)
    - [References](#references)
    - [User Documentation](#user-documentation)
  - [Triage API](#triage-api)
    - [Step 1: Request an authentication token from the Healthwise authorization service](#step-1-request-an-authentication-token-from-the-healthwise-authorization-service)
    - [Step 2: Get list of available Symptom Topics](#step-2-get-list-of-available-symptom-topics)
    - [Step 3: Create/Change Encounter](#step-3-createchange-encounter)
    - [Step 4: Update the Encounter](#step-4-update-the-encounter)
  - [Data Properties](#data-properties)
    - [Action Data Properties](#action-data-properties)
      - [Type: Question](#type-question)
      - [Type: Search](#type-search)
      - [Type: Redirect If the action type is redirect the action data will also have the following properties:](#type-redirect-if-the-action-type-is-redirect-the-action-data-will-also-have-the-following-properties)
      - [Type: Disposition If the action type is disposition the action data will also have the following properties:](#type-disposition-if-the-action-type-is-disposition-the-action-data-will-also-have-the-following-properties)
    - [Response Data Examples:](#response-data-examples)

The Triage API is built and deployed to an Azure Function App via Azure DevOps Pipelines. These pipelines enable us to leverage the latest best practices of CI/CD in development and deployment.

The pipeline is configured via YAML configuration. See [azure-pipelines.yml](./azure-pipelines.yml) for the current configuration. The agent pool used to build the solution is the default Azure Pipelines Hosted Agent.

There are two variables that need to be set on the pipeline:

1. `functionAppName` - This is the name of the resource group that the Triage API will be deployed to.
2. `serviceConnection` - The service account that will be performing the build and deployment.

These variables are located in respective Variable Groups (under: _Pipelines > Library_). The variable groups must be associated with the respective pipeline for them to be leveraged.

The service account can be authorized by creating an **Azure Resource Manager** Service Connection (in: _Project Settings > Service Connections_). Use the advanced link to connect to a previously created service account that has the correct permissions.

### Notes

- Make sure that the build triggers are limited to the branches of interest.
- If creating a personal or experimental pipeline, disable build triggers as they will all fire at the same time leading to the build agent being blocked.
- Each `job` must have an agent `pool` defined or else a provisioning error will be reported.

### References

- [Triage API DevOps Dashboard](https://dev.azure.com/Healthwise-Platform/Platform%20Team/_build?definitionId=25)
- [Azure DevOps Pipelines Documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops)
- [DevOps yaml schema](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema)

### User Documentation

## Triage API 
The Triage API enables you to interact with the check your symptoms logic of Healthwise Symptom Topics through a restful API.  Using a series of guided questions this API helps you identify when someone should seek medical care. Each interaction with the API has two possible responses.  The API will either provide a question or an action. All questions are single answer multiple choice.  An action has two possible outcomes and can either be a recommendation of care or a suggestion to redirect to a more appropriate resource.

The following guide explains the process of using the Triage API. 

### Step 1: Request an authentication token from the Healthwise authorization service

Navigate to [https://auth.healthwise.net/oauth2/token](https://auth.healthwise.net/oauth2/token) and request an authentication token.

```
POST:    https://auth.healthwise.net/oauth2/token
Headers: Authorization Basic {{client.basic}} 
Body:    x-www-form-urlencoded
          grant_type    client_credentials
          scope         triage.api

Returns: Bearer access_token (a jwt, or "Json Web Token") for Triage API access
Example:
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJ....",
    "token_type": "Bearer",
    "expires_in": 86400
}
```

After retrieving an authentication token, you may then start the Triage workflow. A Triage event starts with first creating an encounter. To start an encounter, we first need a list of Healthwise Topics.

### Step 2: Get list of available Symptom Topics

The Triage API encounter endpoint expects a "Healthwise Identifier" or "hwid". To retrieve a list of available topics, first call the "topics" endpoint. This will return a JSON response containing the hwid and the title of the topic.

```
GET:     https://platform.healthwise.net/triage/sx/topics
Headers: Authorization Bearer {{jwt}}
Returns:
Example:
{
    "status": 200,
    "versioning": {
        "X-HW-Version": "1"
    },
    "links": {
        "self": "https://platform.healthwise.net/triage/sx/topics"
    },
    "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.sx.topics.json",
    "data": {
        "topics": [
        {
            "hwid": "aa53422",
            "title": "Change in Heartbeat"
        },
        {
            "hwid": "aa84397",
            "title": "Scalp Problems"
        },
        {
            "hwid": "abdpn",
            "title": "Abdominal Pain, Age 12 and Older"
        },
        ... //Additional Topics// ...
        ]
    }
}
```

We will use the "hwid" values to start an encounter.

### Step 3: Create/Change Encounter

An encounter is a beginning to end session that tracks a user through the requested symptom topic(s). The user is able to switch topics during an encounter. To start an encounter, we need to make a request to the "sx" endpoint. This endpoint takes an "hwid" as a route parameter, and an optional encounter ID (if we are redirecting from another topic.) A new encounter will return the encounter id and the action which contains the first question for the specific symptom topic. See below for more information about the action data properties.

```
POST:    https://platform.healthwise.net/triage/sx/{{hwid}}/{{id}}
Headers: Authorization Bearer {{jwt}}
Body:    Empty
Returns:
Example:
{
  "status": 200,
  "versioning": {
    "X-HW-Version": "1"
  },
  "links": {
    "self": "https://platform.healthwise.net/triage/sx/allre"
  },
  "schema": "https://platform.healthwise.net/spec/schema.healthwise.api.sx.create.json",
  "data": {
    "encounterId": "8dc637f5-e028-4071-a84d-5364a3a5d798",
    "action": {
      "type": "question",
      "question": {
        "id": "1295",
        "type": "confirmation",
        "text": {
          "element": "p",
          "type": "p",
          "attributes": {},
          "content": [
            {
              "element": "$text",
              "content": "Are you concerned about an allergic reaction? Review "
            },
            {
              "element": "xref",
              "type": "a",
              "attributes": {
                "href": "healthrisks"
              },
              "content": [
                {
                  "element": "$text",
                  "content": "health risks"
                }
              ]
            },
            {
              "element": "$text",
              "content": " that may make any symptom more serious."
            }
          ]
        },
        "subText": {},
        "definitions": [
          "healthrisks"
        ],
        "answerSet": [
          {
            "id": "2739",
            "text": "Yes",
            "chartText": "Allergic reaction concerns",
            "confirmation": "C"
          },
          {
            "id": "2740",
            "text": "No",
            "chartText": "Allergic reaction concerns",
            "confirmation": "D"
          }
        ]
      },
      "definitions": {
        "healthrisks": {
          "element": "hwInfoConcept",
          "type": "div",
          "attributes": {
            "id": "healthrisks",
            "hwsxrole": "details"
          },
          "content": [
            {
              "element": "hwInfoConceptBody",
              "type": "article",
              "attributes": {},
              "content": [
                {
                  "element": "section",
                  "type": "section",
                  "attributes": {},
                  "content": [
                    {
                      "element": "p",
                      "type": "p",
                      "attributes": {},
                      "content": [
                        {
                          "element": "$text",
                          "content": "Many things can affect how your body responds to a symptom and what kind of care you may need. These include:"
                        }
                      ]
                    },
                    {
                      "element": "ul",
                      "type": "ul",
                      "attributes": {},
                      "content": [
                        {
                          "element": "li",
                          "type": "li",
                          "attributes": {},
                          "content": [
                            {
                              "element": "hwEmphasis",
                              "type": "strong",
                              "attributes": {},
                              "content": [
                                {
                                  "element": "$text",
                                  "content": "Your age"
                                }
                              ]
                            },
                            {
                              "element": "$text",
                              "content": ". Babies and older adults tend to get sicker quicker."
                            }
                          ]
                        },
                        {
                          "element": "li",
                          "type": "li",
                          "attributes": {},
                          "content": [
                            {
                              "element": "hwEmphasis",
                              "type": "strong",
                              "attributes": {},
                              "content": [
                                {
                                  "element": "$text",
                                  "content": "Your overall health"
                                }
                              ]
                            },
                            {
                              "element": "$text",
                              "content": ". If you have a condition such as diabetes, HIV, cancer, or heart disease, you may need to pay closer attention to certain symptoms and seek care sooner."
                            }
                          ]
                        },
                        {
                          "element": "li",
                          "type": "li",
                          "attributes": {},
                          "content": [
                            {
                              "element": "hwEmphasis",
                              "type": "strong",
                              "attributes": {},
                              "content": [
                                {
                                  "element": "$text",
                                  "content": "Medicines you take"
                                }
                              ]
                            },
                            {
                              "element": "$text",
                              "content": ". Certain medicines, herbal remedies, and supplements can cause symptoms or make them worse."
                            }
                          ]
                        },
                        {
                          "element": "li",
                          "type": "li",
                          "attributes": {},
                          "content": [
                            {
                              "element": "hwEmphasis",
                              "type": "strong",
                              "attributes": {},
                              "content": [
                                {
                                  "element": "$text",
                                  "content": "Recent health events"
                                }
                              ]
                            },
                            {
                              "element": "$text",
                              "content": ", such as surgery or injury. These kinds of events can cause symptoms afterwards or make them more serious."
                            }
                          ]
                        },
                        {
                          "element": "li",
                          "type": "li",
                          "attributes": {},
                          "content": [
                            {
                              "element": "hwEmphasis",
                              "type": "strong",
                              "attributes": {},
                              "content": [
                                {
                                  "element": "$text",
                                  "content": "Your health habits and lifestyle"
                                }
                              ]
                            },
                            {
                              "element": "$text",
                              "content": ", such as eating and exercise habits, smoking, alcohol or drug use, sexual history, and travel."
                            }
                          ]
                        }
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }
      }
    }
  }
}
```

### Step 4: Update the Encounter

As a user answers questions, we need to update the encounter. Depending on the answers of the user, the following actions may be returned.

- Question: The next relevant question to display.
- Search: Based on the last question response, this isn’t the appropriate topic.
- Redirect: Based on the last question response, there is a better topic available.
- Disposition: The recommended outcome

The recommended outcomes are:

- Call Emergency Services Now (emergency)
- Seek Care Now (red) - Seek Care Today (yellow)
- Make an Appointment (black)
- Home Treatment (green)

See the Data Properties section below for an explanation of the different fields; The update encounter endpoint takes the **hwid\*- and **encounter ID\*- as route parameters. You must also include a body with the ID of the question you are answering, and the ID of the answer. These values are found in the previous response.

```
PATCH:   https://platform.healthwise.net/triage/sx/{{hwid}}/{{id}}
Headers: Authorization Bearer {{jwt}}
Body:
    Example:
    {
        "answers": [
            {
                "qId": 1295,
                "rId": 2739
            }
        ]
    }
Response:
Example:
{
  "status": 200,
  "versioning": {
    "X-HW-Version": "1"
  },
  "links": {
    "self": "https://platform.healthwise.net/triage/sx/con11/176088dd-aeff-4eae-b4d4-8ffe0f41805"
  },
  "schema": "https://platform.healthwise.net/spec/schema.healthwise.api.sx.update.json",
  "data": {
    "encounterId": "176088dd-aeff-4eae-b4d4-8ffe0f41805",
    "action": {
      "type": "question",
      "question": {
        "id": "31",
        "type": "age",
        "text": {
          "element": "p",
          "type": "p",
          "attributes": {},
          "content": [
            {
              "element": "$text",
              "content": "How old are you?"
            }
          ]
        },
        "subText": {},
        "definitions": [],
        "answerSet": [
          {
            "id": "72",
            "text": "Less than 12 years",
            "chartText": "Less than 12 years",
            "confirmation": "C",
            "maxAgeMonths": "143"
          },
          {
            "id": "73",
            "text": "12 years or older",
            "chartText": "12 years or older",
            "confirmation": "C",
            "maxAgeMonths": "9999"
          }
        ]
      },
      "definitions": {}
    },
    "previousAnswers": [
      {
        "qId": 182,
        "rId": 411
      }
    ]
  }
}
```

The Update Encounter PATCH endpoint will also return the data property previousAnswers. The previousAnswers is an array with a set of question and response Ids (qId, rId). This data can be used along with the next question and response Id for the body of the next Update Encounter PATCH request.

## Data Properties

The section describes the various response values you will receive.

### Action Data Properties

The action type can be one of the following:

- Question: This is the next relevant question to display.
- Search: Based on the last question response, this isn’t the appropriate topic.
- Redirect: Based on the last question response, there is a better topic available.
- Disposition: Based on all previous question responses, this is the recommended outcome

The possible outcomes are:

- Call Emergency Services Now (emergency)
- Seek Care Now (red)
- Seek Care Today (yellow)
- Make an Appointment (black)
- Home Treatment (green)

#### Type: Question

If the action type is "question", the action block will also the following data properties.

- id: a unique numerical question identifier.
- type: a string that represents the question type; confirmation, age, gender, redirect, trunk, or branch.
- text: a JSON data structure that represents the question text, which may include a reference to a definition.
- subtext: a JSON data structure that represents the question subtext, which may include a reference to a definition.
- definitions: an array of unique definition identifiers referenced in the question.
- answerSet: an array of two or more responses.

A response has the following data properties

- id: a unique numerical response identifier.
- text: string representing the response.
- chartText: summary string for the response.
- confirmation: ‘C’ or ‘D’ Indicates if the response was confirmed or denied.
- maxAgeMonths: age type questions only. The maximum age in months for the response. For example, if the response is “Less than 12 years”, this value will be “143”, which is 11 years 11 months.
- definitions: A question can have zero or more definitions. Each definition has a unique identifier and a JSON data structure that represents the definition text.

A note about definitions: The definition block will contain content that adds content to the current question.

#### Type: Search

If the action type is search the action data will also have the following properties:

- conditions: Returns an array of question and response Ids that led to this search action type.

#### Type: Redirect If the action type is redirect the action data will also have the following properties:

- hwid: This is the topic identifier that the encounter should be redirected to, it will provide a more relevant triage session.
- href: This field provides the complete POST URL to continue the current encounter for the new triage session.
- title: This is the topic title the encounter should be redirected to.
- conditions: Returns an array of question and response Ids that led to this redirect action type.

#### Type: Disposition If the action type is disposition the action data will also have the following properties:

- hwid: This is the recommended disposition outcome identifier.
- disposition: This is a JSON data structure to render specific text related to the disposition.
- conditions: Returns an array of question and response Ids that led to this disposition action type.

### Response Data Examples:

Get Topics - GET

```json
  {
    "status":200,
    "versioning":{
      "X-HW-Version":"1"
    },
    "links":{
      "self":"https://platform.healthwise.net/triage/sx/topics"
    },
    "schema":"https://platform.healthwise.net/triage/spec/schema.healthwise.api.topics.json",
    "data":{
      "topics":[{
        "hwid":" tm7018","title":" Diabetes-Related High and Low Blood Sugar Levels"
      }]
    }
```

Create Encounter - POST

```json
  {
    "status": 200,
    "versioning": {
        "X-HW-Version": "1"
    },
    "links": {
        "self": "https://platform.healthwise.net/triage/sx/tm7018"
    },
    "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.create.json",
    "data": {
        "encounterId": "9c151905-2a43-417c-b43d-a14a91d289fe",
        "action": {
            "type": "question",
            "question": {
                "id": "39",
                "type": "confirmation",
                "text": {
                    JSON Data
                },
                "subText": {
                    //JSON Data//
                },
                "definitions": ["healthrisks"],
                "answerSet": [{
                    "id": "91",
                    "text": "Yes",
                    "chartText": "Diabetes",
                    "confirmation": "C"
                }, {
                    "id": "92",
                    "text": "No",
                    "chartText": "Diabetes",
                    "confirmation": "D"
                }]
            },
            "definitions": {
                "healthrisks": {
                    //JSON Data//
                }
            }
        }
    }
}
```

Update Encounter – PATCH (data.action.type: search)

```json
{
  "status": 200,
  "versioning": {
    "X-HW-Version": "1"
  },
  "links": {
    "self": "https://platform.healthwise.net/triage/sx/tm7018/9c151905-2a43-417c-b43d-a14a91d289fe"
  },
  "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.update.json",
  "data": {
    "encounterId": "9c151905-2a43-417c-b43d-a14a91d289fe",
    "action": {
      "type": "search",
      "conditions": [
        {
          "questionId": 39,
          "responseId": 92,
          "sort": 0
        }
      ]
    },
    "previousAnswers": [
      {
        "qId": 39,
        "rId": 92
      }
    ]
  }
}
```

Update Encounter – PATCH (data.action.type: redirect)

```json
{
  "status": 200,
  "versioning": {
    "X-HW-Version": "1"
  },
  "links": {
    "self": "https://platform.healthwise.net/triage/sx/tm7018/9c151905-2a43-417c-b43d-a14a91d289fe"
  },
  "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.update.json",
  "data": {
    "encounterId": "ce4f8ae5-8acb-4ca3-9a8c-cd3448538305",
    "action": {
      "type": "redirect",
      "hwid": "posto",
      "title": "Postoperative Problems",
      "href": "https://platform.healthwise.net/triage/sx/posto/ce4f8ae5-8acb-4ca3-9a8c-cd3448538305?redirected=shoul",
      "conditions": [
        {
          "questionId": 296,
          "responseId": 655,
          "sort": 0
        },
        {
          "questionId": 497,
          "responseId": 1082,
          "sort": 0
        }
      ]
    },
    "previousAnswers": [
      {
        "qId": 296,
        "rId": 655
      },
      {
        "qId": 168,
        "rId": 382
      },
      {
        "qId": 497,
        "rId": 1082
      }
    ]
  }
}
```

Update Encounter – PATCH (data.action.type: question)

```json
{
  "status": 200,
  "versioning": {
      "X-HW-Version": "1"
  },
  "links": {
      "self": "https://platform.healthwise.net/triage/sx/tm7018/9c151905-2a43-417c-b43d-a14a91d289fe"
  },
  "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.update.json",
  "data": {
      "encounterId": "9c151905-2a43-417c-b43d-a14a91d289fe",
      "action": {
          "type": "question",
          "question": {
              "id": "1365",
              "type": "age",
              "text": {
                  JSON Data
              },
              "subText": {},
              "definitions": [],
              "answerSet": [{
                  "id": "2887",
                  "text": "Less than 3 years",
                  "chartText": "Less than 3 years",
                  "confirmation": "C",
                  "maxAgeMonths": "35"
              }, {
                  "id": "2888",
                  "text": "3 years or older",
                  "chartText": "3 years or older",
                  "confirmation": "C",
                  "maxAgeMonths": "9999"
              }]
          },
          "definitions": {}
      },
      "previousAnswers": [{
          "qId": 39,
          "rId": 91
      }]
  }
}
```

Update Encounter– PATCH(data.action.type: disposition)

```json
{
  "status": 200,
  "versioning": {
      "X-HW-Version": "1"
  },
  "links": {
      "self": "https://platform.healthwise.net/triage/sx/tm7018/9c151905-2a43-417c-b43d-a14a91d289fe"
  },
  "schema": "https://platform.healthwise.net/triage/spec/schema.healthwise.api.update.json",
  "data": {
      "encounterId": "9c151905-2a43-417c-b43d-a14a91d289fe",
      "action": {
          "type": "disposition",
          "hwid": "emergency",
          "disposition": {
              JSON Data
          },
          "conditions": [{
              "questionId": 39,
              "responseId": 91,
              "sort": 0
          }, {
              "questionId": 826,
              "responseId": 1777,
              "sort": 3
          }, {
              "questionId": 827,
              "responseId": 1779,
              "sort": 4
          }]
      },
      "previousAnswers": [{
          "qId": 39,
          "rId": 91
      }, {
          "qId": 1365,
          "rId": 2888
      }, {
          "qId": 826,
          "rId": 1777
      }, {
          "qId": 827,
          "rId": 1779
      }]
  }
}
```
