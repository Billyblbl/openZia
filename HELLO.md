# Hello World example
As the pipeline is already implemented, you will not need a lot of work to get a first "Hello World" example running (meaning more time to develop great modules !).

*Please note that this example is very straightforward and not using advantages of a multithreaded server.*

I assume you have already setup your sever basic's routine (listen, accept, read ...).
Let's say you have a class **Server**.
```C++
    #include <openZia/Pipeline.hpp>
    class Server {
    public:
	    // Set the pipeline module and configuration paths
		Server(void) : _pipeline("ModuleDir", "ConfigDir") {}

	    /* Your routines functions ... */

	private:
		/* Your members ... */
		Pipeline _pipeline; // Pipeline loads module

		// Callback when server receives a message
	    void onPacketReceived(const YourNetworkPacket &packet) {
		    oZ::Context context = buildPipelineContext(packet);
		    _pipeline.runPipeline(context);
		    sendResponseToClient(context);
	    }

		// Helper used to create a pipeline context from a packet
		oZ::Context buildPipelineContext(const YourNetworkPacket&packet) const {
			oZ::Context context;
			/* Parse the HTTP packet and build a context out of it */
			return context;
		}

		void sendResponseToClient(const Context &context) {
			/* Send the HTTP response to the client. You may use the following methods:
				context.hasError() // Fast error checking
				context.getEndpoint() // Get the endpoint of target client
				context.getResponse() // Get the response result of the pipeline
			*/
		}
    };
```

Now you need to create your first module. Let's create two of them as a demonstration of dependency handling :
```C++
/* --- Hello.hpp --- */
#pragma once

#include <openZia/IModule.hpp>

class Hello : public oZ::IModule
{
public:
	Hello(void) = default;
	virtual ~Hello(void) = default;

	virtual const char *getName(void) const { return "Hello"; }
	virtual Dependencies getDependencies(void) const noexcept { return { "World" }; }

	// Register your callback to the pipeline
	virtual void onRegisterCallbacks(Pipeline &pipeline) {
		pipeline.registerCallback(
			oZ::State::Response, // Call at response creation time
			oZ::Priority::Medium + 1, // With medium priority but higher than 'World' module
			this, &Hello::onResponse // Member function style
		);
	}

private:
	void onResponse(oZ::Context &context) {
		oZ::Log(oZ::Information) << "Module 'Hello' wrote successfully its message";
		context.getResponse().getHeader().get("Content-Type") = "text/plain";
		context.getResponse().getBody() += "Hello";
	}
};

/* --- Hello.cpp --- */
#include "Hello.hpp"

extern "C" {
    oZ::ModulePtr CreateModule(void) { return std::make_shared<Hello>(); }
}
```

```C++
/* --- World.hpp --- */
#pragma once

#include <openZia/IModule.hpp>

// Second module 'World'
class World : public oZ::IModule
{
public:
	World(void) = default;
	virtual ~World(void) = default;

	virtual const char *getName(void) const { return "World"; }
	virtual Dependencies getDependencies(void) const noexcept { return { "Hello" }; }

	// Register your callback to the pipeline
	virtual void onRegisterCallbacks(Pipeline &pipeline) {
		pipeline.registerCallback(
			oZ::State::Response, // Call at response creation time
			oZ::Priority::Medium, // With medium priority
			[](oZ::Context &context) { // Lambda function style
				context.getResponse().getBody() += " World";
			})
		);
	}

	virtual void onLoadConfigurationFile(const std::string &directory) {
		/* Load module's configuration if needed, using your own configuration loader */
	}
};

/* --- World.cpp --- */
#include "World.hpp"

extern "C" {
    oZ::ModulePtr CreateModule(void) { return std::make_shared<World>(); }
}
```