---
layout: post
title: "Erlang Tcp Server"
tags:
- erlang
---

### tcp_server_app.erl
```
-module(tcp_server_app).
-behaviour(application).
-export([start/2, stop/1]).
-define(PORT, 2222).

start(_Type, _Args) ->
    io:format("tcp app start~n"),
    case tcp_server_sup:start_link(?PORT) of
        {ok, Pid} ->
            {ok, Pid};
        Other ->
            {error, Other}
    end.

stop(_S) ->
    ok.
```

### tcp_server_sup.erl
```
-module(tcp_server_sup).
-behaviour(supervisor).
-export([start_link/1, start_child/1]).
-export([init/1]).

start_link(Port) ->
    io:format("tcp sup link~n"),
    supervisor:start_link({local, ?MODULE}, ?MODULE, [Port]).

start_child(LSock) ->
    io:format("tcp sup start child~n"),
    supervisor:start_child(tcp_client_sup, [LSock]).

init([tcp_client_sup]) ->
    io:format("tcp sup init client~n"),
    {ok,
    { {simple_one_for_one, 0, 1},
      [
        {tcp_server_handler,
         {tcp_server_handler, start_link, []},
         temporary,
         brutal_kill,
         worker,
         [tcp_server_handler]
         }
       ]
    }
    };

init([Port]) ->
    io:format("tcp sup init~n"),
    {ok,
       {
        {one_for_one, 5, 60},
        [
            {
                tcp_client_sup,
                {
                    supervisor, 
                    start_link,
                    [
                        {
                            local, 
                            tcp_client_sup
                        },
                        ?MODULE,
                        [
                            tcp_client_sup
                        ]
                    ]
                },
                permanent,
                2000,
                supervisor,
                [tcp_server_listener]
            },
            {
                tcp_server_listener,
                {
                    tcp_server_listener, 
                    start_link,
                    [Port]
                },
                permanent,
                2000,
                worker,
                [tcp_server_listener]
            }
        ]
       }
    }.
```

### tcp_server_listener.erl
```
-module(tcp_server_listener).
-behaviour(gen_server).
-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
    terminate/2, code_change/3]).
-record(state, {lsock}).

start_link(Port) ->
    io:format("tcp server listener start ~n"),
    gen_server:start_link({local, ?MODULE}, ?MODULE, [Port], []).

init([Port]) ->
    process_flag(trap_exit, true),
    Opts = [binary, {packet, 0}, {reuseaddr, true},
                {keepalive, true}, {backlog, 30}, {active, false}],
    State =
    case gen_tcp:listen(Port, Opts) of
        {ok, LSock} ->
            start_server_listener(LSock),
            #state{lsock = LSock};
        _Other ->
            throw({error, {could_not_listen_on_port, Port}}),
            #state{}
    end,
    {ok, State}.

handle_call(_Request, _From, State) ->
    io:format("tcp server listener call ~p~n", [_Request]),
    {reply, ok, State}.

handle_cast({tcp_accept, Pid}, State) ->
    io:format("tcp server listener cast ~p~n", [tcp_accept]),
    start_server_listener(State, Pid),
    {noreply, State};

handle_cast(_Msg, State) ->
    io:format("tcp server listener cast ~p~n", [_Msg]),
    {noreply, State}.

handle_info({'EXIT', Pid, _}, State) ->
    io:format("tcp server listener info exit ~p~n", [Pid]),
    start_server_listener(State, Pid),
    {noreply, State};

handle_info(_Info, State) ->
    io:format("tcp server listener info ~p~n", [_Info]),
    {noreply, State}.

terminate(_Reason, _State) ->
    io:format("tcp server listener terminate ~p~n", [_Reason]),
    ok.

code_change(_OldVsn, State, _Extra) ->
    {ok, State}.

start_server_listener(State, Pid) ->
    unlink(Pid),
    start_server_listener(State#state.lsock).

start_server_listener(Lsock) ->
    case tcp_server_sup:start_child(Lsock) of
        {ok, Pid} ->
            link(Pid);
        _Other ->
            do_log
    end.
```

### tcp_server_handler.erl
```
-module(tcp_server_handler).
-behaviour(gen_server).
-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
    terminate/2, code_change/3]).
-record(state, {lsock, socket, addr}).
-define(Timeout, 120*1000).

start_link(LSock) ->
    io:format("tcp handler start link~n"),
    gen_server:start_link(?MODULE, [LSock], []).

init([LSock]) ->
    io:format("tcp handler init ~n"),
    inet:setopts(LSock, [{active, once}]),
    gen_server:cast(self(), tcp_accept),
    {ok, #state{lsock = LSock}}.

handle_call(Msg, _From, State) ->
    io:format("tcp handler call ~p~n", [Msg]),
    {reply, {ok, Msg}, State}.

handle_cast(tcp_accept, #state{lsock = LSock} = State) ->
    {ok, CSock} = gen_tcp:accept(LSock),
    io:format("tcp handler info accept client ~p~n", [CSock]),
    {ok, {IP, _Port}} = inet:peername(CSock),
    start_server_listener(self()),
    {noreply, State#state{socket=CSock, addr=IP}, ?Timeout};

handle_cast(stop, State) ->
    {stop, normal, State}.

handle_info({tcp, Socket, Data}, State) ->
    inet:setopts(Socket, [{active, once}]),
    io:format("tcp handler info ~p got message ~p~n", [self(), Data]),
    ok = gen_tcp:send(Socket, <<Data/binary>>),
    {noreply, State, ?Timeout};

handle_info({tcp_closed, _Socket}, #state{addr=Addr} = State) ->
    io:format("tcp handler info ~p client ~p disconnected~n", [self(), Addr]),
    {stop, normal, State};

handle_info(timeout, State) ->
    io:format("tcp handler info ~p client connection timeout~n", [self()]),
    {stop, normal, State};

handle_info(_Info, State) ->
    io:format("tcp handler info ingore ~p~n", [_Info]),
    {noreply, State}.

terminate(_Reason, #state{socket=Socket}) ->
    io:format("tcp handler terminate ~p~n", [_Reason]),
    (catch gen_tcp:close(Socket)),
    ok.

code_change(_OldVsn, State, _Extra) ->
    {ok, State}.

start_server_listener(Pid) ->
    gen_server:cast(tcp_server_listener, {tcp_accept, Pid}).
```

### tcp_server.app
```
{application, tcp_server,
    [
        {description, "TCP Server"},
        {vsn, "1.0.0"},
        {modules, [tcp_server, tcp_server_app, tcp_server_handler,
                    tcp_server_listener, tcp_server_sup]},
        {registered, []},
        {mod, {tcp_server_app, []}},
        {env, []},
        {applications, [kernel, stdlib]}
    ]
}.
```

### python 测试程序
```
import socket
import struct

s = socket.socket()
host = socket.gethostname()
port = 2222

s.connect((host, port))
s.send(struct.pack("ii", 12, 34))
a1, a2 = struct.unpack("ii", s.recv(1024))
print a1, a2
# print s.recv(1024)
s.close
```