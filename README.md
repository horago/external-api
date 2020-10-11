“Horago External API” (herein after referred to as Horago API or simply API) used for third-patry integrations (POS software, analytics etc). Horago API consists of 2 main parts:
* REST API – used by client applications for querying data (orders, menu items etc), post order changes, post menu updates, etc
* Webhooks – clients application receive updates on certain events, like order payments, status changes, etc

# REST API
Based on [REST architectural style](https://en.wikipedia.org/wiki/Representational_state_transfer) and has the following properties:
* JSON – is a primary serialization format
* HATEOAS – simplifies most of the operations and reduces hardcode level(uses [WebApi.Hal](https://github.com/JakeGinnivan/WebApi.Hal) implementation)
* Authorization - OAuth 2.0
* Documentation - Swagger (Swagger UI)

## HATEOAS
See [WebApi.Hal](https://github.com/JakeGinnivan/WebApi.Hal) for details...

Please note that HATEOAS is enabled only when the following HTTP request header is used: `Accept: application/hal+json`, otherwise response would not contain additional HATEOAS tags.

## Swagger
Swagger UI is available here: https://api.horago.com/swagger. It contains test UI as well as data structure examples.

## Authorization
Forms authorization is used (user credentials must be provided in order to receive API key).

A POST request to
https://api.horago.com/token must be sent with the following:
* HTTP headers: Content-Type: application/x-www-form-urlencoded
* Payload: grant_type=password&username=**&lt;login&gt;**&password=**&lt;password&gt;**

## Workflow
* <a href="#authorization">Login</a> - Post credentials in order to receive authentication token
* Make a GET request to <a href="https://api.horago.com/swagger/ui/index#/Home">Home</a> object. This would return all options available in a form of a HAL links. Only <i>default-place</i> is available at the moment.
* Use the <i>default-place</i> link from the previous step in order to fetch some place information + HAL links to actions available for that particular place:
    * <i>active-orders</i> - can be used to fetch a list of active orders
    * <i>post-menu-items</i> - ...for <a href="#menu-modification">menu modification</a>

# Webhooks
Webhooks provide a simle and liteweight push-style notification splution. Using Webhooks client application can be notified about certain events happening on a server.

## Events
Most of order events triggered by corresponding actions. Action usually also set the certain order flags

Event | Flag | Description
----- | ------ | -----
ping | - | Ping message, client must reply with 2xx HTTP Code
new | - | New Order message
accepted | accepted | Order Accepted
rejected | rejected | Order Rejected
completed | completed | Order Complete
part_completed | partCompleted | Order Partially complete (multiple allowed)
paid | paid | Order has been paid (cash or app)
closed | closed | Order closed (removed from the active list)

## Webhook call validation
Signature verification, helps to verify authenticity of webhook request, as well as prevents replay attacks(under man-in-the-middle conditions). This process involves:
* Request Signature – used in order to prove that request is coming from a trusted source, not a malicious third-party
* Idempotency Key – clients are able to filter out redundant requests
* Request timestamp - may be used to ignore too old requests
### Implementation
A custom HTTP header `x-hwh-signature` would be provided with each request. Its value holds a timestamp and signature in hexadecimal format created using the timestamp and a payload. Example of `x-hwh-signature` HTTP header
`t=1589838888,v1=edefebcaf25439fbb1eb5200740b3842de64db36e7a62fcab0aea3480e58ddce`

In order to check, one must: 
1.	extract the t parameter and v1 parameter from HTTP header
2.	extract actual JSON webhook request payload (i.e., the request body)
3.  extract `idempotency-key` value from HTTP header
3.	concatenate t parameter with idempotency key and with JSON payload using . (dot) as a delimiter
4.	computer HMAC using the SHA256 hash function using the secret key obtained from the place portal
5.	compare the computed HMAC value with the value of v1 parameter. If those match than request is valid
6.	compare the difference between the current timestamp(UTC) and the t value of HTTP header. Decide whether the difference is withing your tolerance window (say 5 mins).  

## Retry logic

Should the original webhook call fail (non-2xx HTTP response code returned or timeout exceeded), the server would be mking several attempts to send the same event again. 
...
HTTP header value `idempotency-key` may be leveraged in order to prevent double-posting of the same event. Client has to remember idempotency keys at least for the period of timestamp window tolerance

## Pings (not finalized)

Server may be sending regular ping events in order to ensure that client is up and ready to process actual events

Timestamp Date format used: https://en.wikipedia.org/wiki/ISO_8601

## Menu modification
HAL link to a batch menu update: **post-menu-items**

The menu may be <a href="https://api.horago.com/swagger/ui/index#!/MenuItems/UpsertItems">updated</a> partially as well as a whole structure. In any case a single POST request is used with the following main attributes:
* <i>categories</i> - array of categories
* <i>options</i> - item option definition with the defaul prices(same options may be shared between several items throuh the optionRefs)
* <i>items</i> - array of products
    * options - non-mandatory array of options
    * catgoryIds - array of category Ids defined previously  
    * optionRefs - arry of references to options defined previously(with the possible price overrides)

* <i>items</i> - array of products with the possibility to define its own set of options if needed

* <i>mode</i> - update strategy

Mode values:
* partialUpdate - may be used to update one or several items. Only new categories added, existing categories would not be changed
* fullUpdate - should be used for a full menu update. Category structure/names would not be changed if modified within Horago previously (new categories may be created), but any items which where not updated would be disabled
* fullRestructure (beta) - the resulting menu would correspond to the request. i.e. category structure/names and child items

## GET Order list example response 
`
{
  "total": 184.4,
  "totalResults": 20,
  "totalPages": 4,
  "page": 1,
  "_links": {
    "self": {
      "href": "/api/v1/places/active-orders?page=1"
    },
    "next": {
      "href": "/api/v1/places/active-orders?page=2"
    },
    "page": {
      "href": "/api/v1/places/{placeId}/active-orders{?page}",
      "templated": true
    },
    "order": [
      {
        "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ecf08c1a64f0e2ec42b9321"
      },
      {
        "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ed05120a64f0e0e28c6835d"
      },
      ...
    ]
  },
  "_embedded": {
    "order": [
      {
        "items": [
          {
            "sides": [
              {
                "name": "Potato",
                "quantity": 1,
                "subtotal": 0.5,
                "options": [
                  {
                    "optId": "5b6414b5012a905420d889b1",
                    "name": "Size",
                    "alts": [
                      {
                        "altId": "3",
                        "name": "Small"
                      }
                    ]
                  },
                  {
                    "optId": "5b6adee1012aa77f18471779",
                    "name": "Some option",
                    "alts": [
                      {
                        "altId": "2",
                        "name": "Alt 2"
                      }
                    ]
                  },
                  {
                    "optId": "5b6adee1012aa77f18471779",
                    "name": "Some option",
                    "alts": [
                      {
                        "altId": "3",
                        "name": "Alt 3"
                      }
                    ]
                  }
                ]
              }
            ],
            "name": "Burito",
            "quantity": 1,
            "subtotal": 4.5,
            "options": [
              {
                "optId": "5b6adee1012aa77f18471779",
                "name": "Some option",
                "alts": [
                  {
                    "altId": "2",
                    "name": "Alt 2"
                  }
                ]
              },
              {
                "optId": "5b6adee1012aa77f18471779",
                "name": "Some option",
                "alts": [
                  {
                    "altId": "3",
                    "name": "Alt 3"
                  }
                ]
              }
            ]
          }
        ],
        "id": "5ecf08c1a64f0e2ec42b9321",
        "isTest": true,
        "created": "2020-05-28T00:41:37.75Z",
        "client": {
          "name": "Oleg Tsilinchuk"
        },
        "currency": {
          "code": "EUR"
        },
        "totalPrice": 5,
        "flags": {
          "paid": true
        },
        "payment": {
          "type": "card",
          "date": "2020-05-29T00:03:00.951Z",
          "tips": 1.075
        },
        "terms": {
          "type": "inhouse",
          "tableNum": "2"
        },
        "etag": "70aee3a66303d808",
        "_links": {
          "self": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ecf08c1a64f0e2ec42b9321"
          },
          "accept": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ecf08c1a64f0e2ec42b9321/actions/accept"
          },
          "reject": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ecf08c1a64f0e2ec42b9321/actions/reject"
          }
        }
      },
      {
        "items": [
          {
            "name": "Burito",
            "quantity": 1,
            "subtotal": 6
          }
        ],
        "id": "5ed05120a64f0e0e28c6835d",
        "isTest": true,
        "created": "2020-05-29T00:03:02.208Z",
        "client": {
          "name": "Oleg Tsilinchuk"
        },
        "currency": {
          "code": "EUR"
        },
        "totalPrice": 6,
        "flags": {
          "paid": true
        },
        "payment": {
          "type": "card",
          "date": "2020-05-29T00:03:00.951Z",
          "tips": 1.075
        },
        "terms": {
          "type": "pickup",
          "when": "2020-05-29T08:00:00Z",
          "notes": "Dijdjdhdj",
          "pickupFrom": {
            "id": "local",
            "name": "Test (Usual flow)"
          }
        },
        "etag": "70aee3a66303d808",
        "_links": {
          "self": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ed05120a64f0e0e28c6835d"
          },
          "accept": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ed05120a64f0e0e28c6835d/actions/accept"
          },
          "reject": {
            "href": "/api/v1/places/5a8eba5f3dc3ac11586b2954/orders/5ed05120a64f0e0e28c6835d/actions/reject"
          }
        }
      ...
      }
    ]
  }
}
`
 
