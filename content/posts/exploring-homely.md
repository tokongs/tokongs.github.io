+++
title = 'Exploring the Homely API with Go'
date = 2024-08-12T21:09:35+02:00
tags = ["Go", "Socket.IO", "Home automation"]
description = "I've played around with The Homely API using Go. Here's what I've found when writing a small client library."
+++

Recently, I moved to a new apartment and "inherited" a subscription to the home alarm service 
[Homely](https://www.homely.no/). This system has a series of smoke detectors, motion sensors, door sensors and a Yale 
door lock. These devices all communicate their state back to Homely through the Zigbee gateway. I also run an instance 
of Home Assitant with Zigbee2MQTT for the rest of my home automation needs. Sadly, the Homly Zigbee network can't 
connect to my Home Assitant network. There is also no other way to integrate these locally, as far as I know at least.  

So, I started searching for a Home Assitant integration. There are some efforts in this 
direction if you follow [this thread in a Norwegian home automation forum.](https://www.hjemmeautomasjon.no/forums/topic/11367-homely-integration/).
Additionally, [github.com/hansrune/homely-tools](https://github.com/hansrune/homely-tools) implements a small application to push
changes from Homely's API to MQTT. The API is currently in beta, and you can get some documentation if you ask their customer support. 
The documentation, however, is lacking. But I guess you can't expect too much of an semi-public beta API. 
Using my language og choice, Go, I did some exploring of their API. The API client I wrote can be found at 
[github.com/tokongs/homely](https://github.com/tokongs/homely). For now, and I suspect maybe forever, the API is read only.

## Authentication

To authenticate against their API they seem to implement the OAuth2 password grant flow. Using this grant 
you can exchange your credentials for a JWT. Just send a POST request to the `https://sdk.iotiliti.cloud/homely/oauth/token` 
endpoint with a payload like `{"username": "<email>", "password": "password"}`. Using the `TokenSource` 
from `golang.org/x/oauth2` we can automatically deal with fetching tokens.

```go
type tokenPayload struct {
	Username string `json:"username"`
	Password string `json:"password"`
}

type tokenResponse struct {
	AccessToken      string    `json:"access_token"`
	ExpiresIn        int       `json:"expires_in"`
	RefreshExpiresIn int       `json:"refresh_expires_in"`
	RefreshToken     string    `json:"refresh_token"`
	TokenType        string    `json:"token_type:"`
	NotBeforePolicy  int       `json:"not-before-policy"`
	SessionState     uuid.UUID `json:"session_state"`
	Scope            string    `json:"scope"`
}

// tokenSource implements golang.org/x/oauth2.TokenSource and 
can be used to create and authenticated http.Client
type tokenSource struct {
	baseURL  string
	username string
	password string
}

func (s *tokenSource) Token() (*oauth2.Token, error) {
	payload := &bytes.Buffer{}

	if err := json.NewEncoder(payload).Encode(tokenPayload{
		Username: s.username,
		Password: s.password,
	}); err != nil {
		return nil, fmt.Errorf("encode token request body: %w", err)
	}

    u := "https://sdk.iotiliti.cloud/homely/oauth/token"
	req, err := http.NewRequest(http.MethodPost, u, payload)
	if err != nil {
		return nil, fmt.Errorf("created token request: %w", err)
	}

	req.Header.Add("Content-Type", "application/json")

	res, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("execute request: %w", err)
	}
	defer res.Body.Close()

	if res.StatusCode >= 300 {
		return nil, fmt.Errorf("get token: %s", res.Status)
	}

	var r tokenResponse

    if json.NewDecoder(res.Body).Decode(r); err != nil {
		return nil, fmt.Errorf("unmarshal token response: %w", err)
	}

	return &oauth2.Token{
		AccessToken:  r.AccessToken,
		TokenType:    r.TokenType,
		RefreshToken: r.RefreshToken,
		Expiry:       time.Now().Add(time.Duration(r.ExpiresIn)),
	}
}
```

With this it's really simple to create an authenticated client like so.
```go
	ts := oauth2.ReuseTokenSource(nil, &tokenSource{
		username: c.Username,
		password: c.Password,
	})

    client := *oauth2.NewClient(context.Background(), ts),
```

## Locations
Now that we can create an authenticated `http.Client` we can fetch resources from Homely. They have an endpoint
`/homely/locations` to list some basic information about each location you have access to, and the more detailed
endpoint `/homely/home/<locationID>`. The request and response types are shown in the snippet below. 

```go
// Location list of loaction
type Location struct {
	Name          string    `json:"name"`
	LocationID    uuid.UUID `json:"locationId"`
	UserID        uuid.UUID `json:"userId"`
	GatewaySerial string    `json:"gatewayserial"`
	PartnerCode   int       `json:"partnerCode"`
}

// Location details from the home endpoint
type LocationDetails struct {
	LocationID         uuid.UUID `json:"locationID"`
	GatewaySerial      string    `json:"gatewayserial"`
	Name               string    `json:"name"`
	AlarmState         string    `json:"alarmState"`
	UserRoleAtLocation string    `json:"userRoleAtLocation"`
	Devices            []Device  `json:"devices"`
}

type Device struct {
	ID           uuid.UUID          `json:"id"`
	Name         string             `json:"name"`
	SerialNumber string             `json:"serialNumber"`
	Location     string             `json:"location"`
	Online       bool               `json:"online"`
	ModelID      uuid.UUID          `json:"modelId"`
	ModelName    string             `json:"modelName"`
	Features     map[string]Feature `json:"features"`
}

type Feature struct {
	States map[string]State `json:"states"`
}

type State struct {
	Value       any       `json:"value"`
	LastUpdated time.Time `json:"lastUpdated"`
}

```

```go
func (c *Client) Locations(ctx context.Context) ([]Location, error) {
	req, err := c.newRequest(ctx, http.MethodGet, "homely/locations", nil)
	if err != nil {
		return nil, fmt.Errorf("new request: %w", err)
	}

	l := []Location{}
	if err := c.doAndDecode(req, &l); err != nil {
		return nil, err
	}

	return l, nil
}

func (c *Client) LocationDetails(ctx context.Context, locationID uuid.UUID) 
    (LocationDetails, error) {
	var l LocationDetails

	req, err := c.newRequest(ctx, http.MethodGet, 
        fmt.Sprintf("homely/home/%s", locationID), nil)
	if err != nil {
		return l, fmt.Errorf("new request: %w", err)
	}

	if err := c.doAndDecode(req, &l); err != nil {
		return l, err
	}

	return l, nil
}
```

As you can see from the types, we can use these endpoints to get the state of all the different sensors of all 
the devices.

## Streaming
Getting the current state of sensor is nice, but what we really want is to be notified of changes so that we 
can use them for automations. Homely's documentation say that this can be done using a Websocket request on 
`//sdk.iotiliti.cloud`. This turned out to have more nuance to it. It is really a Socket.IO endpoint and it lives
on `wss://sdk.iotility.cloud/socket.io/`.

The availablity of Socket.IO clients for Go was surprisingly sparse. I found 
[github.com/googollee/go-socket.io](https://github.com/googollee/go-socket.io), but couldn't get it to work properly
aginst the Homely server. I have no idea, why I couldn't get it to work. I would guess it's possible with more 
intimate knowledge of how the endpoint is configured and exprience using Socket.IO. 

The solution I ended up with was connecting to the endpoint using plain Websockets with 
[github.com/coder/websocket](https://github.com/coder/websocket) and parsing the messages according to the 
[Engine.IO](https://socket.io/docs/v4/engine-io-protocol/) and
[Socket.IO](https://socket.io/docs/v4/socket-io-protocol/) protocols. The process of figuring out 
the small quirks of these protocols was a bit painful as the documentation is sparse.

Engine.IO is a protocol for two way communcation between a client and a server. It can run on top of Websockets or 
HTTP long-polling. Each session starts with a handshake where the client and server sends a couple of messages agreeing
on a session ID and heartbeat parameters. The server will periodically send `ping` messages to which the client has
to resond with `pong` messages to avoid eviction. These messages are called packets and are utf-8 encoded. Each 
packet starts with a packet type identifier and then optionally the packet data. A ping packet is as simple as `2` while
pong is `3`. A packet of the `message` type is encoded like `4I'm_the_message_data`, where `4` is the packet type of a 
message and `I'm_the_message_data` is the message data.

Socket.IO builds on top of Engine.IO and allows sending/receiving from multiple "namespaces". Why these are 2 seperate
protocols, I'm not sure I understand. Someone smarter than me probably found it to be a good idea. Anyway, the Socket.IO 
packets are similar to the Engine.IO packets. Prefixed with a packet type and then the message data. A Socket.IO packet 
sent over Engine.IO might look like `42{"key": "value"}`. This is a Engine.IO packet of the `message` type carrying a 
Socket.IO packet of the `event` type with the JSON object `{"key": "value"}` as the event. 

A Socket.IO session is started by connecting to a namespace. This is done by sending an `open` packet which has the id `0`. 
Connecting to the namespace `/test` can be done with a packet looking like this `40/test`. If you don't specify a namespace
you'll be connected to the `/` namespace.

After going on a several hours long tanget learning a bit about Socket.IO I was finally able to stream some data from the Homely
API. I wrote a small client with partial Socket.IO support. For the Socket.IO server to accept the websocket connection, 
the query parameters `EIO=4` and `transport=websocket` must be set. The API is authenticated using query parameters:scream:.
This is done with the parameter `token=Bearer%20<JWT access token>`. Lastly, the API takes the query parameter 
`location=<locationID>` to designate which location to stream changes from.

```
func (c *Client) HandleEvents(ctx context.Context, h func(name string, msg string) error) error {
	u, err := url.Parse(c.server)
	if err != nil {
		return fmt.Errorf("invalid url: %w", err)
	}

	q := u.Query()
	q.Set("EIO", EngineIOVersion)
	q.Set("transport", Transport)

	if c.tokenSource != nil {
		t, err := c.tokenSource.Token()
		if err != nil {
			return fmt.Errorf("get token: %w", err)
		}

		q.Set("token", fmt.Sprintf("Bearer %s", t.AccessToken))
	}

	u.RawQuery = q.Encode()

	conn, _, err := websocket.Dial(ctx, u.String(), nil)
	if err != nil {
		return fmt.Errorf("websocket dial: %w", err)
	}

	defer func() {
		if err := conn.CloseNow(); err != nil {
			slog.Error("Errored while closing websocket connection", "error", err)
		}
	}()

	// SocketIO open request
	if err := conn.Write(ctx, websocket.MessageText, []byte("40")); err != nil {
		return fmt.Errorf("socketio connect to namespace: %w", err)
	}

	for {
		_, b, err := conn.Read(ctx)
		if err != nil {
			return fmt.Errorf("read: %w", err)
		}

		s := string(b)

		slog.Debug("Got websocket packet", "packet", s)

		// We only care about EngineIO packets. They start with the message type number
		if len(s) < 1 {
			slog.Debug("Packet has no data")
			continue
		}

		eioType, err := strconv.Atoi(string(s[0]))
		if err != nil {
			slog.Debug("Invalid EngineIO type", "type", s[0])
			continue
		}

		if PacketType(eioType) == EIOPacketTypePing {
			slog.Debug("Got EngineIO Ping, will Pong")
			if err := conn.Write(ctx, websocket.MessageText, []byte("3")); err != nil {
				return fmt.Errorf("eio pong: %w", err)
			}

			slog.Debug("Ponged")
			continue
		}

		if len(s) < 2 {
			// it has no data so we don't care
			slog.Debug("Message is not SocketIO message")
			continue
		}

		sioType, err := strconv.Atoi(string(s[1]))
		if err != nil {
			slog.Debug("Invalid SocketIO type", "type", s[1])
			continue
		}

		if PacketType(sioType) != PacketTypeEvent || len(s) < 3 {
			slog.Debug("Skipping non event SocketIO packet")
			continue
		}

		var values []json.RawMessage
		if err := json.Unmarshal([]byte(s[2:]), &values); err != nil {
			slog.Error("Could not unmarshal SocketIO event", "error", err)
			continue
		}

		if len(values) < 2 {
			slog.Error("Got unexpected number of values from SocketIO event", "values", values)
			continue
		}

		var name string
		if err := json.Unmarshal(values[0], &name); err != nil {
			slog.Error("Failed to unmarshal event name", "error", err)
			continue
		}

		data, err := values[1].MarshalJSON()
		if err != nil {
			slog.Error("Failed to handle message body", "error", err)
			continue
		}

		if err := h(name, string(data)); err != nil {
			return err
		}
	}
}
```

Using this I can get a stream of changes that look like this:
```
type Event struct {
	Type string `json:"type"`
	Data EventData `json:"data"`
}

type EventData struct {
	DeviceID       uuid.UUID `json:"deviceId"`
	GatewayID      uuid.UUID `json:"gatewayId"`
	LocationID     uuid.UUID `json:"locationId"`
	ModelID        uuid.UUID `json:"modelId"`
	RootLocationID uuid.UUID `json:"rootLocationId"`
	Changes        []Change  `json:"changes"`
	PartnerCode    int       `json:"partnerCode"`
}

type Change struct {
	Feature     string    `json:"feature"`
	StateName   string    `json:"stateName"`
	Value       any       `json:"value"`
	LastUpdated time.Time `json:"lastUpdated"`
}
```

From what I can see, in my setup, only the temeperature and network connectivity, values are streamed.
One of the main things I wanted yo achieve with the API was to automate my IKEA lights to turn them on 
when my Yale doorlock noticed that it was opened in the evening. So the lack of log outputs in my terminal
when I opened my door was dissapointing. 

## Conclusion
I spent quite a few hours experimenting with this API. While there is value to be gained from the API,
it doesn't really fit my use case. As this is still in beta I hope that they will continue to add new 
features and stream changes from more parts of the system. 

Socket.IO is a weird protocol. Maybe some of the choices make more sense when you spend more time 
understanding the different use cases. The documentation of the protocols is alright, but it's
clear that the expectation is for people to use one of the existing client libraries as it's a bit sparse. 
