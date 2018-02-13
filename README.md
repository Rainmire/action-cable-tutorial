# action-cable-tutorial
A brief look at creating a secure, multi-channel connection to Rails Action Cable

# Overview
If you've found your way here, you're probably looking for a quick and easy way to get Rails to deal with websocket connections. In this brief tutorial, I'll be showing you how to send instant messages from the Rails server to your frontend. The examples I'll be featuring uses React, but you should be able to connect this to any similar frontend framework.

# What is Action Cable?
Action Cable is Rails 5's solution to dealing with websocket connections. It utilizes a pub/sub paradigm to broadcast information to groups of subscribers. If you want to know more about the nitty-gritty details, here's the official documentation: http://guides.rubyonrails.org/action_cable_overview.html

# Step 1: Connecting to `/cable`
The first step in the connection process lies with `cable.js`, a handy JavaScript file Rails 5 generates for you by default. Here's what mine looks like.
```
//= require action_cable
//= require_self
//= require_tree ./channels

(function() {
  this.App || (this.App = {});

  App.cable = ActionCable.createConsumer();

}).call(this);
```
This is what establishes the connection between the client and the server via `/cable`. It also gives you `App.cable` on which you can call methods to create and destroy subscriptions, receive messages, and do basically everything else in the frontend. Here's an example of an `setSocket` action creator to handle this connection.
```
export const setSocket = () => (dispatch) => {
  return App.cable.subscriptions.create({
    channel: 'ChatChannel'
  }, {
    connected: () => {console.log(`connected to ChatChannel`);},
    disconnected: () => {console.log(`disconnected from ChatChannel`);},
    received: (data) => {
      dispatch(parseMessage(data.message));
    }
  });
};
```
Here I create a subscription to a channel named ChatChannel. This name is important, so remember it for later. There are callbacks provided for connecting and disconnecting, but it's fine to leave them empty. `received` is where you'll be listening to broadcasts from the server. `data` contains all the information that was received. My data contains a `message`, so I pull that out and dispatch it elsewhere to be processed. How you handle this part is up to you.

And that's really it for the frontend. Simple right?

# Step 2: Authenticating the Connection
If your app requires the user to be logged in before using the socket connection, this step will check for that. Otherwise, move on to Step 3.