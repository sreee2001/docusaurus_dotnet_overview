### SignalR

#### Short Introduction

SignalR is a real-time communication library that enables bi-directional communication between client and server, automatically choosing the best transport method.

#### Official Definition

ASP.NET Core SignalR is an open-source library that simplifies adding real-time web functionality to apps, enabling server-side code to push content to clients instantly.

#### Usage

```csharp
// Hub
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserJoined", $"{Context.UserIdentifier} joined {groupName}");
    }

    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserLeft", $"{Context.UserIdentifier} left {groupName}");
    }

    public override async Task OnConnectedAsync()
    {
        await Clients.All.SendAsync("UserConnected", Context.UserIdentifier);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await Clients.All.SendAsync("UserDisconnected", Context.UserIdentifier);
        await base.OnDisconnectedAsync(exception);
    }
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chathub");
```

#### Client-side (JavaScript)

```javascript
// wwwroot/js/chat.js
const connection = new signalR.HubConnectionBuilder()
  .withUrl("/chathub")
  .build();

// Start connection
connection
  .start()
  .then(function () {
    document.getElementById("sendButton").disabled = false;
  })
  .catch(function (err) {
    console.error(err.toString());
  });

// Receive messages
connection.on("ReceiveMessage", function (user, message) {
  const li = document.createElement("li");
  li.textContent = `${user}: ${message}`;
  document.getElementById("messagesList").appendChild(li);
});

// Send message
document
  .getElementById("sendButton")
  .addEventListener("click", function (event) {
    const user = document.getElementById("userInput").value;
    const message = document.getElementById("messageInput").value;
    connection.invoke("SendMessage", user, message).catch(function (err) {
      console.error(err.toString());
    });
    event.preventDefault();
  });
```

#### Blazor SignalR Client

```csharp
@page "/chat"
@using Microsoft.AspNetCore.SignalR.Client
@inject NavigationManager Navigation
@implements IAsyncDisposable

<div class="form-group">
    <label>
        User:
        <input @bind="userInput" />
    </label>
</div>
<div class="form-group">
    <label>
        Message:
        <input @bind="messageInput" @onkeypress="@(async (e) => { if (e.Key == "Enter") await Send(); })" />
    </label>
</div>
<button @onclick="Send" disabled="@(!IsConnected)">Send</button>

<hr>

<ul id="messagesList">
    @foreach (var message in messages)
    {
        <li>@message</li>
    }
</ul>

@code {
    private HubConnection? hubConnection;
    private List<string> messages = new();
    private string userInput = string.Empty;
    private string messageInput = string.Empty;

    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/chathub"))
            .Build();

        hubConnection.On<string, string>("ReceiveMessage", (user, message) =>
        {
            var encodedMsg = $"{user}: {message}";
            messages.Add(encodedMsg);
            InvokeAsync(StateHasChanged);
        });

        await hubConnection.StartAsync();
    }

    private async Task Send()
    {
        if (hubConnection is not null)
        {
            await hubConnection.SendAsync("SendMessage", userInput, messageInput);
            messageInput = string.Empty;
        }
    }

    public bool IsConnected =>
        hubConnection?.State == HubConnectionState.Connected;

    public async ValueTask DisposeAsync()
    {
        if (hubConnection is not null)
        {
            await hubConnection.DisposeAsync();
        }
    }
}
```

#### Use Cases

- Real-time chat applications
- Live dashboards
- Gaming applications
- Collaborative tools
- Live notifications
- Stock price updates
- Social media feeds

#### When to Use / When Not to Use

**Use SignalR when:**

- Need real-time communication
- Bi-directional data flow required
- Building collaborative applications
- Live updates are essential

**Consider alternatives when:**

- Simple request/response patterns
- Batch data processing
- One-way communication sufficient
- High scalability requirements

#### Market Alternatives

- Socket.IO (Node.js)
- WebSockets (native)
- Server-Sent Events (SSE)
- gRPC streaming

---
