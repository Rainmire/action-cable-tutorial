# action-cable-tutorial
A brief look at creating a secure, multi-channel connection to Rails Action Cable

# Overview
If you've found your way here, you're probably looking for a quick and easy way to get Rails to deal with websocket connections. In this brief tutorial, I'll be showing you how to send instant messages from the Rails server to your frontend. The examples I'll be featuring uses React, but you should be able to connect this to any similar frontend framework. If you want to check out my project that this tutorial's examples come from, you can find it here: https://github.com/Rainmire/message-me

# What is Action Cable?
Action Cable is Rails 5's solution to dealing with websocket connections. It utilizes a pub/sub paradigm to broadcast information to groups of subscribers. If you want to know more about the nitty-gritty details, here's the official documentation: https://github.com/rails/rails/tree/master/actioncable

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
Here I create a subscription to a channel named `ChatChannel`. This name is important, so remember it for later. There are callbacks provided for connecting and disconnecting, but it's fine to leave them empty. `received` is where you'll be listening to broadcasts from the server. `data` contains all the information that was received. My data contains a `message`, so I pull that out and dispatch it elsewhere to be processed. How you handle this part is up to you.

And that's really it for the frontend. Simple right?

# Step 2: Authenticating the Connection
If your app requires the user to be logged in before using the socket connection, this step will check for that. Otherwise, move on to Step 3.

Much like how ApplicationController is the parent class of every controller, Connection is the parent class of every Channel you create. When the client initiates a socket connection with Rails, an instance of Connection is created. This socket connection can be validated by adding a bit of code to `connection.rb`.
```
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
 
    def connect
      self.current_user = find_verified_user
    end
 
    private
    def find_verified_user
      token = cookies.encrypted[:session_token]
      if token && verified_user = User.find_by(session_token: token)
        verified_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```
The connect method is called automatically when the client, also called the consumer, attempts to establish a connection. We have `connect` call the helper method `find_verified_user` to validate the user for us. Here's the catch though: you can't call the `session` hash in the Connection class. At least, I haven't been able to find a way to invoke `session[:session_token]` or some similar method, since it appears to not be available in this scope (probably because Connection does not inherit from `ActionController::Base`). What I did to work around this issue is to use `cookies.encrypted` in my session validation instead.
```
def login(user)
  user.reset_session_token!
  # session[:session_token] = user.session_token
  cookies.encrypted[:session_token] = user.session_token

  @current_user = user
end
```
I've commented out the old session hash that I used before. Rails `session` by default stores its data as a cookie on the client side anyway, so I took a similar approach and encrypted the `session_token` in a cookie to be delivered to the client. `cookies.encrypted` ensures that the cookie is both unreadable and tamper-proof (except by the server of course). If you do this, be sure to change ALL instances of `session` to `cookies.encrypted`, and not just the one shown here! We'll then search for the user among those who are currently logged in, and return either `verified_user` or `reject_unauthorized_connection`, both of which are conveniently provided for us by Rails.

One more thing. We can allow all channel classes derived from this parent to access the current user's identity with `identified_by :current_user`. That allows us to fetch any relevant information for this particular user later on.

# Step 3: Streaming from Channels
Within your `channels` directory, (where the `application_cable` directory lies) you can create channel classes of `#{name}_channel`. I have one called `chat_channel`, which handles all the messages sent between users.
```
class ChatChannel < ApplicationCable::Channel
  def subscribed
    memberships = current_user.conversation_memberships
    memberships.each do |membership|
      stream_from "chat_#{membership.conversation_id}"
    end
  end
end
```
Remember `ChatChannel` from Step 1? This is where it comes in play. By subscribing to `ChatChannel`, the user will be able to to stream particular broadcasts defined for that user in the `subscribed` method of the `ChatChannel` class. My users all have `conversation_memberships`, and because we have access to the `current_user` defined earlier, we can use that information to determine which broadcasts this particular user should be allowed to listen to. The logic for this class will vary depending on your needs, of course.

# Step 4: Broadcasting

For this step, let's start with a Message model. What this model will be depends on what kind of information you want to broadcast to users. This is what an example model might look like,
```
class Message < ApplicationRecord
  after_commit { MessageRelayJob.perform_later(self) }

  belongs_to :conversation
  belongs_to :author,
  class_name: :User,
  foreign_key: :author_id
  ...
end
```
Notice the first line in that class. This line triggers every time a Message object is saved to the database. It calls a job that will handle broadcasting, passing itself along as the parameter.
```
class MessageRelayJob < ApplicationJob
  def perform(message)
    id = message.conversation_id
    message = Api::MessagesController.render(
      partial: 'api/messages/message',
      locals: { message: message }
    )
    ActionCable.server.broadcast("chat_#{id}",
                                 message: JSON.parse(message))
  end
end
```
Inside the job class, the message to be sent is constructed using a partial view. It is then broadcasted to the channel `chat_#{id}`, which is received by `stream_from "chat_#{membership.conversation_id}"` back in Step 3. It's certainly not necessary to use a partial to do this. In fact, `message` can be as simple as a string.

## That's it!
That's everything you need to get a basic channel subscription working using Rails Action Cable. You can even add additional channel classes if you need to send different types of information.