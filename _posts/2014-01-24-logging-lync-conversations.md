---
layout: post
title: "Logging Lync conversations"
date: 2014-01-24 22:52:25 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/01/24/logging-lync-conversations/
---

A feature missing in Lync is the ability to log chat conversations to a text file. There is an option to log conversations to the "Conversation History" folder in Outlook, but this option can be disabled by an administrator. I like being able view and search my conversation history. Since this option will be disabled soon, I need to find a way to continue logging my conversations. Fortunately for me, I can use the Lync SDK to my advantage.

Each time a chat or video session is started, they are encapsulated within a container. These containers are known as `Conversations` in Lync.

{% raw %}
```csharp
LyncClient client = LyncClient.GetClient();

client.ConversationManager.ConversationAdded += 
	(sender, eventArgs) => Console.WriteLine("Chat window opened.");

client.ConversationManager.ConversationRemoved += 
	(sender, eventArgs) => Console.WriteLine("Chat window closed.");
```
{% endraw %}

Conversations can contain multiple modes or "modalities". In my case, I'm only interested in logging instant messages. On first try, my code looked similar to this:

{% raw %}
```csharp
client.ConversationManager.ConversationAdded += (sender, eventArgs) =>
{
	var instantMessageModality = 
		(InstantMessageModality) eventArgs.Conversation.Modalities[ModalityTypes.InstantMessage];

	instantMessageModality.InstantMessageReceived += (o, data) =>
	{
		var mode = (InstantMessageModality) o;
		var name = (string) mode
			.Participant
			.Contact
			.GetContactInformation(ContactInformationType.DisplayName);

		Console.WriteLine("Received a message from: " + name);
		Console.WriteLine("The message received is: " + data.Text);
	};
};
```
{% endraw %}

I noticed the console was only printing messages from myself. When a message was received from another person, nothing happened. It turns out that I need to create an event handler for each person involved in the conversation. Since I was the original person who opened the chat window, the participant was set to myself.

Since multiple chat windows can be opened at the same time, we need to create a handler for each conversation window. Within each conversation window, we need to create a handler for each participant. For each participant, we need to create a handler to handle any messages we receive.

{% raw %}
```csharp
// For each conversation
client.ConversationManager.ConversationAdded += (cSender, cEventArgs) =>
{
	// For each participant
	cEventArgs.Conversation.ParticipantAdded += (pSender, pEventArgs) =>
	{
		var modality = (InstantMessageModality) pEventArgs
			.Participant
			.Modalities[ModalityTypes.InstantMessage];

		// Register for messages
		modality.InstantMessageReceived += (mSender, mEventArgs) =>
		{
		};
	};
};
```
{% endraw %}

At this point, we have access to the person sending the message and the message text. We can append it to a file or log it to a database.

{% raw %}
```csharp
// Register for messages
modality.InstantMessageReceived += (mSender, mEventArgs) =>
{
	var instantMessageModality = (InstantMessageModality) mSender;
	var person = (string) instantMessageModality
		.Participant
		.Contact
		.GetContactInformation(ContactInformationType.DisplayName);

	using (var conn = new SQLiteConnection(@"data source=D:\Lync\messages.db"))
	using (var cmd = conn.CreateCommand())
	{
		conn.Open();

		cmd.CommandText = 
			@"insert into messages (conversationid, date, person, message) 
			values (@conversationid, @date, @person, @message)";
			
		cmd.Parameters.AddWithValue("@conversationId", conversationId);
		cmd.Parameters.AddWithValue("@date", DateTime.Now);
		cmd.Parameters.AddWithValue("@person", person);
		cmd.Parameters.AddWithValue("@message", mEventArgs.Text);

		cmd.ExecuteNonQuery();
	}
};
```
{% endraw %}
