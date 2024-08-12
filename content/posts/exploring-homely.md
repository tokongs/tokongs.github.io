+++
title = 'Exploring the Homely API with Go'
date = 2024-08-12T21:09:35+02:00
draft = true
+++

Recently, I moved to a new appartment and "inherited" a subscription to the home alarm service 
[Homely](https://www.homely.no/). This system has a series of fire alarms, motion sensors, door sensors and a Yale 
door lock. These devices all communicate their state back to Homely through the Zigbee gateway. I also run an instance 
of Home Assitant with Zigbee2MQTT for the rest of my home automation needs. Sadly, the Homly Zigbee network can't 
connect to my Home Assitant network. There is also no other way to integrate these locally, as far as I know at least.  

So, I started searching for Home Assitant integration. There are some efforts going into this 
direction if you follow [this thread in a Norwegian home automation forum.](https://www.hjemmeautomasjon.no/forums/topic/11367-homely-integration/),
but that's about the most I can find about how to use Homely's API. The API is currently in beta, and you can get
some documentation if you ask their customer support. The documentation, however, is lacking. But I guess you can't
expect too much of an semi-public beta API. Using my language og choice, Go, I did some exploring of their API.
The API client I wrote can be found in [this repo](https://github.com/tokongs/homely). For now, and I suspect maybe 
forever, the API is read only.

## Authentication

To authenticate against their API they seem to implement the OAuth2 password grant flow. To get a JWT, you can send a 
POST request to the `https://sdk.iotiliti.cloud/homely/oauth/token` endpoint with a payload like
`{"username": "<email>", "password": "password"}`. Using the `TokenSource` from `golang.org/x/oauth2` we can automatically
deal with fetching tokens.

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
Now that we can create an authenticated `http.Client` we can fetch resources from Homely. They have one endpoint
`/homely/locations` to list some basic information about each location you have access to and the more detailed
endpoint `/homely/home/<locationID>`. The request and response types are encoded below. 

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
`wss//sdk.iotiliti.cloud`. This turned out to have more nuance to it. It is really a Socket.IO endpoint and it lives
on `wss://sdk.iotility.cloud/socket.io/`.

The availablity of Socket.IO clients for Go was surprisingly sparse. I found 
[github.com/googollee/go-socket.io](https://github.com/googollee/go-socket.io), but couldn't get it to work. 
I have no idea, why I couldn't get it to work. I would guess it's possible with more intimate knowledge of how the
endpoint is configured and exprience using Socket.IO. 

The solution I ended up with was connecting to the endpoint using plain Websockets with 
[github.com/coder/websocket](https://github.com/coder/websocket) and parsing the messages according to the 
[Engine.IO](https://socket.io/docs/v4/engine-io-protocol/) and
[Socket.IO](https://socket.io/docs/v4/socket-io-protocol/) protocols. It did end up in a lot of pain figuring out 
small quirks of these protocols.

Engine.IO is a protocol for two way communcation between a client and a server. It can run on top of Websockets or 
HTTP long-polling. Each session starts with a handshake where the client and server sends a couple of messages agreeing
on a session ID and heartbeat parameters. The server will periodically send `ping` messages to which the client had
to resond with `pong` messages to not be kicked out. These messages are called packets and are utf-8 encoded. Each 
packet stars with a packet type identifier and then optionally the packet data. A ping packet is as simple as `2` while
pong is `3`. A packet of the `message` type is encoded like `4I'm_the_message_data`, where `4` is the packet type of a 
message.

Socket.IO builds on top of Engine.IO and allows sending/receiving from multiple "namespaces". Why these are 2 seperate
protocols, I'm not sure I understand. Someone smarter than me probably found that to be a good idea. 
