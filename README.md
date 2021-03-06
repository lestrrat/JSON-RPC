# NAME

JSON::RPC - JSON RPC 2.0 Server Implementation

# SYNOPSIS

    # app.psgi
    use strict;
    use JSON::RPC::Dispatch;

    my $dispatch = JSON::RPC::Dispatch->new(
        prefix => "MyApp::JSONRPC::Handler",
        router => Router::Simple->new( ... )
    );

    sub {
        my $env = shift;
        $dispatch->handle_psgi($env);
    };

# DESCRIPTION

JSON::RPC is a set of modules that implement JSON RPC 2.0 protocol.

    If you are using old JSON::RPC code (up to 0.96), DO NOT EXPECT
    YOUR CODE TO WORK WITH THIS VERSION. THIS VERSION IS 
    ****BACKWARDS INCOMPATIBLE****

# BASIC USAGE

The JSON::RPC::Dispatch object is responsible for marshalling the request.

    my $dispatch = JSON::RPC::Dispatch->new(
        router => ...,
    );

The routing between the JSON RPC methods and their implementors are handled by
Router::Simple. For example, if you want to map method "foo" to a "MyApp::JSONRPC::Handler" object instance's "handle\_foo" method, you specify something like the following in your router instance:

    use Router::Simple::Declare;
    my $router = router {
        connect "foo" => {
            handler => "+MyApp::JSONRPC::Handler",
            action  => "handle_foo"
        };
    };

    my $dispatch = JSON::RPC::Dispatch->new(
        router => $router,
    );

The "+" prefix in the handler classname denotes that it is already a fully qualified classname. Without the prefix, the value of "prefix" in the dispatch object will be used to qualify the classname. If you specify it in your Dispatch instance, you may omit the prefix part to save you some typing:

    use JSON::RPC::Dispatch;
    use Router::Simple::Declare;

    my $router = router {
        connect "foo" => {
            handler => "Foo",
            action  => "process",
        };
        connect "bar" => {
            handler => "Bar",
            action => "process"
        }
    };
    my $dispatch = JSON::RPC::Dispatch->new(
        prefix => "MyApp::JSONRPC::Handler",
        router => $router,
    );

    # The above will roughly translate to the following:
    #
    # for method "foo"
    #    my $handler = MyApp::JSONRPC::Handler::Foo->new;
    #    $handler->process( ... );
    #
    # for method "bar"
    #    my $handler = MyApp::JSONRPC::Handler::Bar->new;
    #    $handler->process( ... );

The implementors are called handlers. Handlers are simple objects, and will be instantiated automatically for you. Their return values are converted to JSON objects automatically.

You may also choose to pass objects in the handler argument to connect in  your router. This will save you the cost of instantiating the handler object, and you also don't have to rely on us instantiating your handler object.

    use Router::Simple::Declare;
    use MyApp::JSONRPC::Handler;

    my $handler = MyApp::JSONRPC::Handler->new;
    my $router = router {
        connect "foo" => {
            handler => $handler,
            action  => "handle_foo"
        };
    };

# HANDLERS

Your handlers are objects responsible for returning some sort of reference structure that can be properly encoded via JSON/JSON::XS. The handler only needs to implement the methods that you specified in your router.

The handler methods will receive the following parameters:

    sub your_handler_method {
        my ($self, $params, $procedure, @extra_args) = @_;

        return $some_structure;
    }

In most cases you will only need the parameters. The exact format of the $params is dependent on the caller -- you will be passed whatever JSON structure that caller used to call your handler.

$procedure is an instance of JSON::RPC::Procedure. Use it if you need to figure out more about the procedure.

@extra\_args is optional, and will be filled with whatever extra arguments you passed to handle\_psgi(). For example, 

    # app.psgi
    sub {
        $dispatch->handle_psgi($env, "arg1", "arg2", "arg3");
    }

will cause your handlers to receive the following arguments:

    sub your_handler_method {
        my ($self, $params, $procedure, $arg1, $arg2, $arg3) = @_;

    }

This is convenient if you have application-specific data that needs to be passed to your handlers.

# EMBED IT IN YOUR WEBAPP

If you already have a web app (and whatever framework you might already have), you may choose to embed JSON::RPC in your webapp instead of directly calling it in your PSGI application.

For example, if you would like to your webapp's "rpc" handler to marshall the JSON RPC request, you can do something like the following:

    package MyApp;
    use My::Favorite::WebApp;

    sub rpc {
        my ($self, $context) = @_;

        my $dispatch =  ...; # grab it from somewhere
        $dispatch->handle_psgi( $context->env );
    }

# ERRORS

When your handler dies, it is automatically included in the response hash, unless no response was requested (see ["NOTIFICATIONS"](#notifications)).

For example, something like below 

    sub rpc {
        ...
        if ($bad_thing_happend) {
            die "Argh! I failed!";
        }
    }

Would result in a response like

    {
        error => {
            code => -32603,
            message => "Argh! I failed! at ...",
        }
    }

However, you can include custom data by die()'ing with a hash:

    sub rpc {
        ...
        if ($bad_thing_happend) {
            die { message => "Argh! I failed!", data => time() };
        }
    }

This would result in:

    {
        error => {
            code => -32603,
            message => "Argh! I failed! at ...",
            data => 1339817722,
        }
    }

# NOTIFICATIONS

Notifications are defined as procedures without an id.
Notification handling does not produce a response. When all procedures are notifications no content is returned (if the request is valid).
To maintain some basic compatibility with relaxed client implementations, JSON::RPC::Dispatch includes responses when procedures do not have a "jsonrpc" field set to "2.0".

Note that no error is returned in response to a notification when the handler dies or when the requested method is not available.

For example, a request structure like this:

    [
        {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
        {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
        {"jsonrpc": "2.0", "method": "keep_alive"},
        {"jsonrpc": "2.0", "method": "get_data", "id": "9"}
    ]

Would result in a response like

    [
        {"jsonrpc": "2.0", "result": 7, "id": "1"},
        {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}
    ]

# BACKWARDS COMPATIBILITY

Eh, not compatible at all. JSON RPC 0.xx was fine, but it predates PSGI, and things are just... different before and after PSGI.

Code at version 0.96 has been moved to JSON::RPC::Legacy namespace, so change your application to use JSON::RPC::Legacy if you were using the old version.

# AUTHORS

Daisuke Maki

Shinichiro Aska

Yoshimitsu Torii

# AUTHOR EMERITUS

Makamaka Hannyaharamitu, <makamaka\[at\]cpan.org> - JSON::RPC modules up to 0.96

# COPYRIGHT AND LICENSE

The JSON::RPC module is

Copyright (C) 2011 by Daisuke Maki

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself, either Perl version 5.8.0 or,
at your option, any later version of Perl 5 you may have available.

See JSON::RPC::Legacy for copyrights and license for previous versions.
