# JSON-WS
=======

## Overview

JSON-WS (for JSON Web Services, aka JSON-RPC) is a library for creating and running web service/RPC-enabled applications.  
The underlying messaging is based on the [JSON-RPC 2.0] (http://json-rpc.org/wiki/specification "RPC 2.0 Specification") protocol specs.  

JSON-WS allows implementors to expose their APIs (existing or new) over some transport mechanism (HTTP, WebSocket), and it enables clients to make RPC calls to these APIs.  
A set of proxy generators automatically creates client code for Javascript, Java, and C# which can be used to communicate with a JSON-WS service.

## Features

- JSON-RPC v2.0 messages are supported over HTTP (REST), WebSocket, and/or Socket.IO transports
- Method namespacing (e.g. vray.start)
- Group support (useful for documentation purposes)
- Event support
- Versioning support
- Wrap/expose all methods of an object as a service
- Lean type system, including a number of built-in types
- Externally imported types
- Autogenerated metadata page which displays nicely the service API, types, events, etc.
- Autogenerated proxy code for JavaScript (browsers and Node.JS), Java, C#, Python (coming soon)
- Javascript playground along with code snippets and examples
- Registry service - lists all services attached to the registry

## Type system

The JSON-WS library sports a lean type system which allows the developer to overcome the overly dynamic nature of JavaScript and communicate to the world using strongly-typed interfaces.

At their disposal, developers have the ability to use both built-in types, as well as user-defined types (enums and structures - which are similar to a C structure).

Types can be described in external files using JSON notation and can later be imported into a service definition. This approach also allows type definitions to be shared between multiple services in a system.

All user-defined types are exported and recreated in the target client language when a proxy generator is used (e.g. in Java, C#, or Python).  
JavaScript is exempt from this rule -- JavaScript client proxies do *NOT* have user-defined types generated for them; instead they simply use the raw JSON objects passed over the wire.

#### Built-in types

A number of built-in types are available by default:
- **any** - represents any JavaScript value; this is the most dynamic of all types
- **int/integer** - represents a integer value; floating point numbers will be auto-truncated to an integer (no rounding will be applied)
- **number/float/double** - represents a floating point value
- **date/time** - represents a date value; the service's code will recieve an instance of the Date class in Node.JS when using this type; clients will receive either a proper date class instance, if the target framework (like Java or .NET) supports it, or an ISO-formatted date string (e.g. 2013-12-14T11:00:53.379Z)
- **bool/boolean** - represents a boolean value; the service's code is guaranteed that values of this type will be either true or false
- **object/json** - represents a dynamic JSON structure/value
- **string** - represents a string value
- **url** - represents a URL; the service's code will receive an instance of the Url class in Node.JS when using this type; clients will receive either a proper URL class instance if the target framework (e.g. Java or .NET) supports it, or the URL value as a string
- **buffer/binary** - represents raw binary data; the service's code will receive an instance of the Buffer class in Node.JS; clients will recieve the a byte array if the target framework supports it (e.g. Java or .NET), or an instance of the Buffer class (JavaScript - Node.JS and browsers).

#### User-defined types

User-defined types can be of two flavours: enums, or structures.

##### Enums

Enums represent a set of valid values. Enum members are represented by strings, but they also have a numeric value. For example, an enum can be defined as:

	{
		Production: -1,
		RtCpu: 0,
		RtGpuOpenCL: 4,
		RtGpuCUDA: 5
	}
Or, it can be defined as an array of string literals:
	
	['and', 'or']
	
When the latter syntax is used the numeric values of each enum member are defined as the ordinal position of this member: 'and' \== 0, 'or' \== 1.

##### Structures

Structures represent complex hierarchical types, their closest match being C structures.  
Each structure has members, or fields, of a given type. A field's type can either be built-in or user-defined, and it can be a single value or an array of values of the given type.  
There is no limit on the number of the levels of hierarchies that may be used in a user-defined type.

An example structure would be defined as follows:

	{
		width: 'int',
		height: 'int',
		name: 'string',
		ids: ['int'] // array of integers,
		dynamic: 'any',
		tree: 'object', // tree structure encoded as a JSON object
		image: 'binary'
	}

## API

The following sections will describe the APIs exported by the JSON-WS library. The following definition will be assumed from this point on:

	var jsonws = require('json-ws');

### Creating and configuring a service

To create a service, use the *jsonws.api* method which creates an API builder:

	/**
	 * API constructor/builder
	 * @param {String} version The API version tag.
	 * @param {String} friendlyName A string which identifies the API.
	 */
	var api = jsonws.api('version', 'friendlyName');

This method creates a service API builder using the following parameters:

- *version*: specifies the version of this API instance; it can be an arbitrary string, but preferably it is something meaningful (e.g. 1.0.5); this string later becomes part of the HTTP URL that is used to access the service
- *friendlyName*: specifies the name of the API; this string will be displayed in various places, including documentation

### Set the current namespace

The JSON-WS library supports *namespaces*. That is, you can group your RPC endpoints (only methods and events are eligible). E.g. all V-Ray-related functionality can be places in the vray namespace:

	api.namespace('vray');
	api.define('start', ...);
	
This way there will be a *start* method in the *vray* namespace.  
Namespaces may be nested by using dot notation:

	api.namespace('n1.sub1.sub2');

Namespaces generate nesting in the client proxy's code, implemented using sub-classes (in languages like Java and C#) or nested objects (JavaScript). This results in the following example usage (JavaScript):

	// On the server
	api.namespace('vray');
	api.define('start', ...);
	
	// On the client
	var proxy = new Proxy('http://service-location');
	proxy.vray.start();

The default namespace is the empty string, i.e. the root of the service. If, after having used namespaces in your code, you wish to switch to the default one again, simply call:
	
	api.namespace(); /* Switch to the default namespace */
	api.namespace(''); /* or call with an empty string */

### Set the current group

Groups are provided for convenience when displaying the service API documentation. A group is a container for anything defined in the service including types, enums, methods, and events.

There is a default group named 'Default'. To switch to another group, call *api.group*:

	api.group('Some group name', 'Group description');
	
You can provide a descriptive text for the group, but it's optional. If a group with the specified name doesn't exist one will be created.  
The current group is the most recently used one. To switch to another group, call the *group* method again.

### Define enums

	/**
	 * Defines an enum using a set of values
	 * @param {String} enumName The name of the new enum.
	 * @param values A JSON structure or an array of string literals which holds the enum's values .
	 * @param {String} [description] Enum descriptive text.
	 * @returns {{type: String, convert: Function}}
	 */
	api.enum(enumName, values, description);

The values can be either a JSON structure or an array of string literals (please see the example given above).  
When only the first argument is given, this method will try to return the object which stores the internal definition of the enum, if one is found.

When a method expectes an enum value as input, it will always be given the string literal which corresponds to an enum member (after successful validation).To convert the string literal to its numeric value do this:

	function someMethod(enumValue: SomeEnumType) {
		var enumType = api.enum('SomeEnumType');
		// enumType = api.type('SomeEnumType');
		var numericEnumValue = enumType.struct[enumValue];
	}

### Define types

	/**
	 * Registers a type/enum or returns its metadata.
	 * @param {String} typeName The full name of the type.
	 * @param [typeDef] A type/enum definition object.
	 * @param {String} [description] Descriptive text.
	 * @param {Boolean} [isEnum] Flag indicating if we are registering an enum vs. an ordinary type (structure).
	 * @returns {{type: String, convert: Function}}
	 */
	api.type(typeName, typeDef, description, isEnum)
	
The type definition object is a JSON structure which describes every field of the type. Each field has at the least a name and a type. Also, a field can be optional, and a default value may be assigned to it. Finally, a field's type can be described as a single value or an array. Optionally, each field can have a description. Examples:

	var typeStruct = {
		width: 'int' // simplest way to declare a required field
		arrayOfInts: ['int'], // simplest way to declare a required field as an array of elements
		
		// Full declaration listing all supported options
		manyOptions: {
			type: 'SomeCustomType',
			description: 'This is an example field',
			required: false,
			default: {
				// In a real-world case this object
				// will describe a valid value for the 'SomeCustomType' type
			}
		}
	};
	
If *api.type* is called only with a type name, it will return the type metadata if such a type has already been registered.

**Note:** Enums are a special case of types. The *isEnum* flag is internal and reserved for use by the *api.enum* method. Its use is discouraged.

**Note:** Once defined, a type cannot be overriden/redefined. Any attempt to call *api.type* twice in an attempt to redefine a type will throw an exception.

### Define events

	/**
	 * Defines an event that can be emitted to clients.
	 * @param {String} eventName the name of the event (e.g. 'imageReady')
	 * @param options A JSON structure with the event options, or a string holding descriptive text.
	 * @returns {api} The current API instance.
	 */
	api.event(eventName, options);
	
Every service API instance (created using *jsonws.api*) is also an EventEmitter. However, before using an event so it can reach a service's clients, it has to be registered.

Events can have a type, but are not required to. Such events serve as notifications. If you would like to include data with each event declare a type and assign it to your event.

When raised/emitted, an event is sent to all clients who have subscribed for this event. Subscribing/unsubscribing is performed by sendind a special message to the service. Its method name is *rpc.on* (for subscribing) or *rpc.off* (for unsubscribing). The client proxies handle this automatically using the event subscription mechanism of the target language.

Examples of event definitions and use:

	api.event('imageUpdated'); // Notification event, it has no type
	api.emit('imageUpdated');
	
	api.event('imageReady', 'Descriptive text'); // options as a string will set a description
	
	api.event('fullOptionsEvent', {
		type: 'int',
		description: 'Descriptive text'
	});
	api.emit('fullOptionsEvent', 42); // pass in event data as the second argument to emit
	
The same rules apply to the type of a typed event as when a type's field is declared: the type specified must have been already registered, and it optionally can be an array. E.g. to declare an event which sends an array of strings:

	api.event('logUpdated', {
		type: ['string']
	})

### Define RPC endpoints

Defining RPC endpoints (or simply, methods) is the main purpose of this library.

	/**
	 * Defines an RPC endpoint (method).
	 * @param {*|String} options Either a string (the method's name) or an options object.
	 * @param {String} options.name The name of the method.
	 * @param {String} [options.description] Text describing the method.
	 *
	 * @param [options.params] An array of JSON structures, each of which describes an input parameter
	 * @param options.params.name The parameter's name.
	 * @param options.params.type The parameter's type. If omitted 'any' is assumed.
	 * @param options.params.default Optional default value for the parameter. If specified, the parameter becomes optional.
	 * @param options.params.description Descriptive text for the parameter.
	 *
	 * @param {String} options.returns The method's return type, or 'async' if the method doesn't return a value, but still wants to notify its caller when its activity has finished.
	 * @param {*} [options.this] An optional object that will be used as the 'this' object pointer during the RPC call.
	 *
	 * @param {Function} [fn] The function that will be called when an client makes an RPC call to this method.
	 *
	 * @returns {api} The current API instance.
	 */
	api.define(options, fn);


####  Automatically expose all methods of an object on the API:

	/**
	 * Exposes methods of an object on the API
	 * @param {*} obj An object whose functions will be mapped on top of the API.
	 * @param {String[]} [methodNames] A list of method names to include from the target object.
	 * @returns {api} The current API instance.
	 */
	api.defineAll(someObject, whiteList);

The second optional argument is a list with the names of the methods to be defined (the rest are skipped).

	api.defineAll(someObject, ['f1', 'f2', 'f4']);

It is possible to override a method which has been defined:

	someObject.f1 = function(par1) { ...};
	api.defineAll(someObject);
	api.define({'name': 'f1', 'params': [{ 'name': 'par1', 'type': 'string' }], 'help': '...'}); // specify the params and provide help 

#### Access defined methods from inside the API:

Use the *fn* property to access methods:

	api.define('f1', function(f1){ ... });
	api.define('f2', function f2() {
		api.fn.f1(); // call already defined method f1()
	});


#### Set the current 'this' object pointer:

`this()` sets the 'this' object pointer for all methods defined afterwards
(until another call of `this()`).

	api.group('Some group').this(someObject).defineAll(someObject);
	// Override the method 'f1' of someObject:
	api.define('f1', function f1() {
		console.log('calling method f1');
		this.f1();  // use the 'this' object pointer
	});

### Import definitions from external sources

### Code snippets and examples

### Attach transports and listen for requests

	var transport = require('json-ws').transport;
	var app = require('express')();
	var srv = http.createServer(app).listen(app.get('port'), ...);
	api.transport(new transport.HTTP(srv, app))
		.transport(new transport.WebSocket(srv))
		.listen('/thePathToListen');

The address *'rootAddress/thePathToListen/apiVersion'* must be used to open a
WebSocket connection or send an http request to the API server.

Default value for the parameter of `listen()` is '' (the empty string). The api *version* is as specified in the constructor or defaults to '0.0.1'.

#### Close all transports:

	api.close();



## Call/consume the service from clients

The server url:  *rootAddress/thePathToListen/apiVersion* serves:

- metadata describing all methods and events - name, prototype, help

- list of available proxies (with autogenerated proxy code)

(example: "localhost:3000/endpoint/1.0")

The server url:  *rootAddress/thePathToListen/apiVersion?json* serves the api method map in JSON format.

#### Subscribe for events:

	{
		jsonrpc: '2.0',
		method: 'rpc.on',       // 'rpc.off' to unsubscribe
		params: ['testEvent1', 'testEvent2']
	}

#### Example for http request:

	require('request').post({
		url : 'http://localhost:3000/endpoint/1.0',
		json : {
			method: 'sum',
			params: {b: 1, a: 2}, // params: [2, 1]
			id: 1			
		}
	}, function(error, response, body) {...});
	
#### Example for websocket client:

	var WebSocket = require('ws');
	var ws = new WebSocket('ws://localhost:3000/endpoint/1.0');
	ws.on('open', function() {
		ws.send(JSON.stringify({
			  jsonrpc: '2.0',
			  id: 1,
			  method: 'sum',
			  params: [2, 1]
		}));

		ws.send(JSON.stringify({
		  jsonrpc: '2.0',
		  method: 'rpc.on',
		  params: ['testEvent']
		}));
	});

#### Autogenerated proxy code

The proxy code can be used for convenience - instead of manually generating and sending messages or requests,
we can call methods of the proxy object. 

	require('json-ws').proxy(url, function(err, proxy) {});
The parameter *'url'* needs to specify the type of *'proxy'* (language) and optionally a *'localName'* (default is 'Proxy').

	var jsonws = require('json-ws');
	jsonws.proxy('http://localhost:3000/endpoint/1.0?proxy=JavaScript&localName=Tester', function(err, proxy){
		if (err) {
			console.log(err);
			return;
		}
		var a = new proxy.Tester();  // create proxy object
		a.namespace1.sum(1, 1, function(err, result) {  // call a method of this object
			console.log(result);
		});	
		a.on('testEvent', function(data) {...}); // subscribe to an event
		a.removeAllListeners('testEvent'); // unsubscribe
	});


 
