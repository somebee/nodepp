#nodepp

An EPP implementation in node.js

##Description

A _service_ for communicating with EPP registries(ars).

### What it is

A service for communication with registries over EPP. It
takes datastructures in JSON, converts them to XML, sends them to the
registry, and then does the whole thing in reverse with the response. You
should get back something in JSON format.

You also do not have to login. The app logs into the registry when it is
started.



## Installation


1. Clone the repository: `git clone git@github.com:ideegeo/nodepp.git`.
2. `cd nodepp`
3. `npm install` to install node dependencies.

5.  ```source nodepp.rc``` to include ```./node_modules/.bin``` in the
    ```$PATH```. This is necessary for testing or if you plan on running it as
    a daemon.

## Configuration
At the moment, I recommend copying `config/epp-config-template.json` to
something like `config/epp-config-devel.json` or
`config/epp-config-production.json` and and modifying them to fit your needs.
This includes adding your own login/password as well as paths to any SSL certs
you may need.

You can add as many registries as you like. Note that you can add the same
registry multiple times with different logins, etc. This is practical for
testing if you want to simulate logging in as separate registrars for
transfers.

When you've got the config setup the way you like it, you will need to
symlink this to `config/epp-config.json` to run the application.

ln -s `pwd`/config/epp-config-devel.json `pwd`/config/epp-config.json



## Testing

        npm test

## Running the web service

You can start the process trivially using for example:

    node lib/server.js -r registry1

This will start a single epp client that is logged into "Registry1".

    foreverd start -o nodepp-stout.log -e nodepp-sterr.log lib/server.js \
        -r registry1 -r registry2 -r registry3

This runs it as  daemon in the background.  This tells it to open connections
to three different registries.


I recommend using the program **Postman** which can be installed in
Chrome/Firefox as an extension. However, you can also use curl or the
scripting language of your choice. I put an example of this down below.

## RabbitMQ

There is also a RabbitMQ interface:

    node lib/rabbitpee.js -r hexonet-test1

To run it as a daemon:

    foreverd start -o nodepp-stout.log -e nodepp-sterr.log lib/rabbitpee.js \
        -r hexonet-test1 -r nzrs-test1 -r nzrs-test2

*Note* that for all commands documented below, the datastructure that is sent
via RabbitMQ needs to be modified as follows:

     {
        "command": "<command name>",
        "data": <request data>
     }

Data returned by the service should be the same.

For demo purposes, I've written a small script ```lib/rabbitpoo.js``` to send
a single rpc command to a *running* service.  It's hardcoded to send a command
for the ```hexonet-test1``` test accreditation. You can copy/modify it
accordingly to experiment with other registry accreditations as they
become available.


## Not running the service

Kill the daemon process with:

    foreverd stop lib/server.js

When running locally, ```Ctrl-C``` will do.

Sending ```kill -INT``` to the ```pid``` of the ```lib/server.js``` process
triggers the client to send a ```logout``` command to the registries
and then shutdown. You can use this to kill local processes instead of
```Ctrl-C```.  With the daemon, this has the net effect of causing it to
```logout``` and then ```login``` again since ```foreverd```  will
automatically restart the process. This might be handy if some registries have
connection time limits.


Since I haven't gotten around to generating a pid file, the following command
is useful.


    kill -INT `ps waux | grep server.js | grep -v grep | awk '{print $2}'`


## Commands

At the time of this writing, the following commands have been implemented:


### checkContact

```javascript
{"id": "P-12345xyz"}
```

or

```javascript
{"contact": "P-12345xyz"}
```


### infoContact

```javascript
{"id": "P-12345xyz"}
```

or


```javascript
{"contact": "P-12345xyz"}
```



### createContact


```javascript
{
    "id": "my-id-1234",
    "voice": "+1.9405551234",
    "fax": "+1.9405551233",
    "email": "john.doe@null.com",
    "authInfo": {
        "pw": "xyz123"
    },
    "postalInfo": [{
        "name": "John Doe",
        "org": "Example Ltd",
        "type": "int",
        "addr": [{
            "street": ["742 Evergreen Terrace", "Apt b"],
            "city": "Springfield",
            "sp": "OR",
            "pc": "97801",
            "cc": "US"
        }]
    }]
}
```

Some registries set the ```id``` by default. In such cases it's common to use
```auto```. The value for ```type``` may also vary for different
registries. Some require ```loc``` and some require ```int```. EPP allows for
up to 2 _postaInfo_ entries, however I've never seen a registry that accepts
more than 1. For that reason, you can just specify it as a single object:

```javascript
"postalInfo": {
    "name": "John Doe",
    "org": "Example Ltd",
    "type": "int",
    "addr": [{
        "street": ["742 Evergreen Terrace", "Apt b"],
        "city": "Springfield",
        "sp": "OR",
        "pc": "97801",
        "cc": "US"
    }]
}
```

It will be passed to the registry in the appropriate format. The same applies
to the _addr_ field (in _postalInfo_), which can also be specified as an Array
or single object.

```javascript
"addr": {
    "street": ["742 Evergreen Terrace", "Apt b"],
    "city": "Springfield",
    "sp": "OR",
    "pc": "97801",
    "cc": "US"
}
```

### updateContact


```javascript
{
    id: "p-12345",
    add: ['clientDeleteProhibited'],
    rem: ['clientTransferProhibited'],
    chg: {
        "postalInfo": [{
            "name": "John Doe",
            "org": "Example Ltd",
            "type": "loc",
            "addr": [{
                "street": ["742 Evergreen Terrace", "Apt b"],
                "city": "Eugene",
                "sp": "OR",
                "pc": "97801",
                "cc": "US"
            }]
        }],
        "voice": "+1.9405551234",
        "fax": "+1.9405551233",
        "email": "john.doe@null.com",
        "authInfo": {
            "pw": "xyz123"
        },
        "disclose": {
            "flag": 0,
            "disclosing": ["voice", "email"]
        }
    }
}
```


### checkDomain

The following are equivalent:

```javascript
{"domain": "something.com"}
```

or

```javascript
{"name": "something.com"}
```

It is possible to check more than one domain at a time.


```javascript
        {"domain": ["test-domain.com", "test-domain2.com", "test-domain3.com"]}
```


### infoDomain


```javascript
        {"domain": "something.com"}
```


In case you are wondering if you can send multiple domains like in
_checkDomain_, the answer is no. That's not possible in EPP. The result that
you will get back in one _infoDomain_ will be complicated enough.

### createDomain

```javascript
{
    "name": "iwmn-test-101-domain.com",
    "period": {
        "unit": "y",
        "value": 1
    },
    "ns":[
        "ns1.hexonet.net",
        "ns2.hexonet.net"
    ],
        "registrant": "my-id-1234",
    "contact": [
        { "admin": "my-id-1235" }, 
        { "tech": "my-id-1236" }, 
        {"billing": "my-id-1236"} 
    ],
    "authInfo": {
        "pw": "Axri3k.XXjp"
    }
}
```


### transferDomain

```javascript
{
    "name": "test-domain.com",
    "op": "request",
    "period": 1,
    "authInfo": {
        "roid": "P-12345", // optional
        "pw": "2fooBAR"
    }
}
```

Valid values for ```op``` are _approve_, _cancel_, _query_, _reject_, and
_request_. There uses are:

   * Requesting side
      1. _request_ to request a transfer.
      2. _cancel_ to cancel a transfer.
      3. _query_ to find out if a transfer is pending (although we should get
         info via polling)


   * Domain holder side
      1. _approve_ to approve a transfer request from another registrar.
      2. _reject_ to reject a transfer request from another registrar.

### updateDomain

```javascript
{
    "name": "test-domain.com",
    "add": {
        "ns": ["ns3.test.com", "ns4.whatever.com"],
        "contact": [{
            "admin": "P-9876"
        },
        {
            "billing": "PX143"
        }],
        "status": ["clientUpdateProhibited", {
            "s": "clientHold",
            "lang": "en",
            "value": "Payment Overdue"
        }]
    },
    "rem": {
        "ns": [{
            "host": "ns1.test-domain.com",
            "addr": {
                "type": "v4",
                "ip": "192.68.2.132"
            }
        }],
        "contact": [{
            "billing": "PX147"
        }],
        "status": ["clientTransferProhibited", {
            "s": "clientWhatever",
            "lang": "en",
            "value": "Payment Overdue"
        }]
    },
    "chg": {
        "registrant": "P-49023",
        "authInfo": {
            "pw": "TestPass2"
        }
    }
}
```

This is a very complicated example but at least shows what is possible in an
_updateDomain_. At least 1 of ```add```, ```rem```, or ```chg``` is required.
The ```chg``` field, if provided, must contain either a ```registrant```
and/or an ```authInfo```. ```add``` and ```rem``` elements, if provided, must
contain any one or more ```ns```, ```contact```, or ```status``` fields.



## General stuff


Some of the required datastructures might seem a bit weird. EPP has a fairly
complex grammar that is _probably_ intended to make granular control of
domain related entities possible. There are no flat datastructures and some
things must be specified explicitly that would be assumed in systems like
the Hexonet API. For example, to remove nameservers from a domain, it is
necessary to remove them explicitly. Simply updating domain with _the new
nameservers_ will not work. The same goes for contact objects.

### Host objects

In _createDomain_ and _updateDomain_ I've tried to account for 2 different
types of host objects. In the simplest version you can just pass an array of
strings:

```javascript
    ["ns1.host.com", "ns2.host.co.nz"]
```

In cases where IP addresses are required, the following variants can be used:

```javascript
    [{host: "ns1.host.com", addr: "62.47.23.1"}]
    [{host: "ns2.host.com", addr:[ "62.47.23.1", {ip: "53.23.1.3"}    ]}]
    [{host: "ns3.host.com", addr:[ {ip: "::F3:E2:23:::", type: "v6"}, {ip:"47.23.43.1", type: "v4"} ]}]
```

```type``` is ```v4``` by default. You'll have to specify ```v6``` explicitly for IPv6
addresses.

It's up to you to know which cases glue records are required. This
implementation has no way to know that.

### authInfo

_createContact_, _createDomain_, _transferDomain_ and _updateDomain_ accept an
```authInfo``` parameter.

Following are equivalent:

```javascript
    authInfo: "te2tP422t"
```

or

```javascript
    authInfo: {
        pw: "te2tP422t"
    }
```

In some cases you may need to supply a ```roid``` in addition to the
```authInfo```.  This is used to identify the registrant or contact object if
and only if the given authInfo is associated with a registrant or contact
object, and not the domain object itself.

```javascript
authInfo: {
        pw: "te2tP422t",
        roid: "P-1234"
}
```


### period


The ```period``` argument in _createDomain_ and  _transferDomain_ can be specified as follows:

1 year registration

```javascript
period: 1
```

24 month registration

```javascript
period: {
    unit: "m",
    value: 24
}
```

The default unit is _y_ for year and default period is 1.

### transactionId

A ```transactionId``` is optional. It can be added at the top level of the JSON data
structure. By default it will be set to ```iwmn-<epoch timestamp>```.

### Extensions

You can optionally add an ```extension``` property to some commands. This
varies from registry to registry like everything else.  A good example is when
adding ```DNSSEC``` data to a *createDomain*:


```javascript
{
    "name": "iwmn-test-101-domain.com",
    "period": {
        "unit": "y",
        "value": 1
    },
    "ns":["ns1.hexonet.net","ns2.hexonet.net"],
        "registrant": "my-id-1234",
    "contact": [
        { "admin": "my-id-1235" },
        { "tech": "my-id-1236" },
        {"billing": "my-id-1236"}
    ],
    "authInfo": {
        "pw": "Axri3k.XXjp"
    },
    "extension": {
        "DNSSEC": {
            "maxSigLife": 604800,
            "dsData": {
                "keyTag": 12345,
                "alg": 3,
                "digestType": 1,
                "digest": "49FD46E6C4B45C55D4AC",
                "keyData": {
                    "flags": 257,
                    "protocol": 3,
                    "alg": 1,
                    "pubKey": "AQPJ////4Q=="
                }
            }
        }
    }
}
```

As stated earlier, this implementation doesn't know anything about _business
logic_ for any of the registries. The data given is just converted to XML and
sent on its way. Actually determining what goes into the extension data needs
to be done at a higher level. In the case of DNSSEC that would mean that
signing, algorithm and key info needs to come from the caller.

### DNSSEC

I've implemented ```DNSSEC``` EPP generation for create and update. I don't
know if we'll ever need this. NZRS has it in their extension list, but I
didn't see any examples of it anywhere. Hexonet only references it with their
KV interface.

Following are some variations that you can send (I'm leaving out the standard
part of the EPP request):

#### createDomain

Create a domain with the ```dsData``` interface:

```javascript
"extension": {
    "DNSSEC": {
        "maxSigLife": 604800,
        "dsData": {
            "keyTag": 12345,
            "alg": 3,
            "digestType": 1,
            "digest": "49FD46E6C4B45C55D4AC"
        }
    }
}
```

with the ```keyData``` interface:

```javascript
"extension": {
    "DNSSEC": {
        "keyData":{
            "flags": 257,
            "protocol": 3,
            "alg": 1,
            "pubKey": "AQPJ////4Q=="
        }
    }
}
```

with the ```keyData``` in the ```dsData``` element:

```javascript
"extension": {
    "DNSSEC": {
        "maxSigLife": 604800,
        "dsData": {
            "keyTag": 12345,
            "alg": 3,
            "digestType": 1,
            "digest": "49FD46E6C4B45C55D4AC",
            "keyData":{
                "flags": 257,
                "protocol": 3,
                "alg": 1,
                "pubKey": "AQPJ////4Q=="
            }
        }
    }
}
```

#### updateDomain


Add a ```dsData``` key and remove a ```keyData``` key and change the
```maxSigLife``` of the key

```javascript
"extension": {
    "DNSSEC": {
        "add": {
            "dsData": {
                "keyTag": 12345,
                "alg": 3,
                "digestType": 1,
                "digest": "49FD46E6C4B45C55D4AC"
            }
        },
        "rem": {
            "keyData": {
                "flags": 257,
                "protocol": 3,
                "alg": 1,
                "pubKey": "AQPJ////4Q=="
            }
        },
        "chg": {
            "maxSigLife": 604800
        }
    }
}
```

Remove all existing key info and replace it with something new:

```javascript
"extension": {
    "DNSSEC": {
        "rem": {
            "all": true
        },
        "add": {
            "dsData": {
                "keyTag": 12345,
                "alg": 3,
                "digestType": 1,
                "digest": "49FD46E6C4B45C55D4AC"
            }
        }
    }
}
```


## Example usage:

Post the following to http://localhost:3000/command/hexonet/checkDomain

    prompt$ time curl -H "Content-Type: application/json" \
        -d '{"domain": "test-domain.com"}'  \
                http://localhost:3000/command/hexonet/checkDomain

_Note_ I just put ```time``` in there to show what the execution time is.
*YMMV*


You should get the following response (or something similar):

        "result":{
            "code":1000,"msg":"Command completed successfully"},
            "resData":{
                "domain:chkData": {
                    "xmlns:domain":"urn:ietf:params:xml:ns:domain-1.0",
                    "xsi:schemaLocation":"urn:ietf:params:xml:ns:domain-1.0 domain-1.0.xsd",
                "domain:cd":{
                    "domain:name":{
                        "avail":0,
                        "$t":"test-domain.com"
                        },
                    "domain:reason":"Domain exists"
                    }
                }
            },
        

I plan to get rid of some of the EPP cruft in the near future.
