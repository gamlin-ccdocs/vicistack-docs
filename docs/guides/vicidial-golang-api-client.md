# Building a VICIdial API Client in Go: Real Code, Real Examples

**VICIdial's API is powerful but its documentation reads like it was written by someone who hated HTTP. This guide builds a complete Go client library — lead management, real-time monitoring, campaign control — with error handling, connection pooling, and tests. Copy the code, hit the API, move on with your life.**

---

## Why Go for VICIdial Integration

You can talk to VICIdial's API from any language. Python, PHP, Node, bash with curl — all work fine. We chose Go for our production integrations for three specific reasons:

1. **Concurrency.** When you're polling 200 agents' real-time status every 5 seconds, Go's goroutines handle concurrent HTTP requests without the thread pool management headaches you'd get in Python or the callback soup you'd get in Node.

2. **Single binary deployment.** Compile once, copy the binary to your VICIdial server, run it. No Python virtual environments, no Node modules, no dependency resolution. Our monitoring daemon is a single 8MB binary.

3. **Performance.** Not that API calls to VICIdial are CPU-intensive, but when you're processing thousands of lead records or aggregating real-time data for dashboards, Go's speed means your tooling never becomes the bottleneck.

That said, if you're a Python shop, translate the concepts. The API calls are the same regardless of language. The Go-specific stuff — goroutines, channels, interfaces — is just how we structure the concurrency.

---

## Understanding VICIdial's API

VICIdial exposes two APIs, and the distinction matters:

### Non-Agent API

URL: `https://your-dialer.com/vicidial/non_agent_api.php`

This handles operations that don't require an agent session — lead management, list operations, campaign configuration, callback scheduling. It's the one you'll use for backend integrations, CRM syncing, and lead import automation.

Authentication is via query parameters: `user` and `pass` in every request. Not headers. Not tokens. Query parameters. Yes, really. Use HTTPS.

### Agent API

URL: `https://your-dialer.com/agc/api.php`

This controls agent-facing operations — login, logout, pause, resume, dial, transfer, hangup. It requires an active agent session and communicates using the agent's credentials.

Both APIs return data as plain text (not JSON, not XML — plain text with delimiters). The response format is:

```
SUCCESS: function - result_data
```
or
```
ERROR: function - error_message
```

Some functions return pipe-delimited fields. Some return newline-separated records. Some return a mix. There's no consistent format across all endpoints. This is the part that makes building a client library worth doing — you parse it once and never think about it again.

---

## Project Setup

```bash
mkdir vicidial-go
cd vicidial-go
go mod init github.com/yourorg/vicidial-go
```

---

## The Core Client

This is the foundation. Every API call flows through this client struct.

```go
// client.go
package vicidial

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"strings"
	"time"
)

// Client is the VICIdial API client.
type Client struct {
	baseURL    string
	user       string
	pass       string
	httpClient *http.Client
	source     string // identifies the API consumer in VICIdial logs
}

// Config holds client configuration.
type Config struct {
	// BaseURL is the VICIdial server URL (e.g., "https://dialer.example.com")
	BaseURL string

	// User is the API-enabled VICIdial user
	User string

	// Pass is the API user's password
	Pass string

	// Timeout is the HTTP request timeout (default: 30s)
	Timeout time.Duration

	// Source identifies this API consumer in VICIdial logs (default: "GO_API")
	Source string
}

// NewClient creates a new VICIdial API client.
func NewClient(cfg Config) *Client {
	timeout := cfg.Timeout
	if timeout == 0 {
		timeout = 30 * time.Second
	}
	source := cfg.Source
	if source == "" {
		source = "GO_API"
	}

	return &Client{
		baseURL: strings.TrimRight(cfg.BaseURL, "/"),
		user:    cfg.User,
		pass:    cfg.Pass,
		source:  source,
		httpClient: &http.Client{
			Timeout: timeout,
			Transport: &http.Transport{
				MaxIdleConns:        100,
				MaxIdleConnsPerHost: 100,
				IdleConnTimeout:    90 * time.Second,
			},
		},
	}
}

// APIError represents an error returned by the VICIdial API.
type APIError struct {
	Function string
	Message  string
	RawBody  string
}

func (e *APIError) Error() string {
	return fmt.Sprintf("vicidial API error [%s]: %s", e.Function, e.Message)
}

// doRequest executes an API request and returns the raw response body.
func (c *Client) doRequest(ctx context.Context, endpoint string,
	params map[string]string) (string, error) {

	u, err := url.Parse(c.baseURL + endpoint)
	if err != nil {
		return "", fmt.Errorf("invalid URL: %w", err)
	}

	q := u.Query()
	q.Set("user", c.user)
	q.Set("pass", c.pass)
	q.Set("source", c.source)
	for k, v := range params {
		q.Set(k, v)
	}
	u.RawQuery = q.Encode()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, u.String(), nil)
	if err != nil {
		return "", fmt.Errorf("creating request: %w", err)
	}
	req.Header.Set("User-Agent", "vicidial-go/1.0")

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return "", fmt.Errorf("executing request: %w", err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("reading response: %w", err)
	}

	rawBody := strings.TrimSpace(string(body))

	// Check for API-level errors
	if strings.HasPrefix(rawBody, "ERROR:") {
		parts := strings.SplitN(rawBody, " - ", 2)
		function := strings.TrimPrefix(parts[0], "ERROR: ")
		message := ""
		if len(parts) > 1 {
			message = parts[1]
		}
		return "", &APIError{
			Function: function,
			Message:  message,
			RawBody:  rawBody,
		}
	}

	return rawBody, nil
}

// nonAgentAPI makes a request to the Non-Agent API.
func (c *Client) nonAgentAPI(ctx context.Context,
	params map[string]string) (string, error) {
	return c.doRequest(ctx, "/vicidial/non_agent_api.php", params)
}

// agentAPI makes a request to the Agent API.
func (c *Client) agentAPI(ctx context.Context,
	params map[string]string) (string, error) {
	return c.doRequest(ctx, "/agc/api.php", params)
}
```

The connection pooling configuration in `http.Transport` is important. VICIdial's API doesn't do keepalive negotiation, but the underlying TCP connections benefit from reuse when you're making many requests in sequence (like polling agent status).

---

## Lead Management

Lead operations are the most common API integration. Import leads from your CRM, update dispositions, search for existing records.

```go
// leads.go
package vicidial

import (
	"context"
	"fmt"
	"strings"
)

// Lead represents a VICIdial lead record.
type Lead struct {
	LeadID      string
	ListID      string
	PhoneNumber string
	PhoneCode   string
	FirstName   string
	LastName    string
	Address1    string
	City        string
	State       string
	PostalCode  string
	Country     string
	Email       string
	Status      string
	Comments    string
	// VICIdial supports many more fields — add them as needed
}

// AddLeadResult is returned after successfully adding a lead.
type AddLeadResult struct {
	LeadID  string
	Message string
}

// AddLead adds a single lead to VICIdial.
func (c *Client) AddLead(ctx context.Context, lead Lead) (*AddLeadResult, error) {
	params := map[string]string{
		"function":     "add_lead",
		"phone_number": lead.PhoneNumber,
		"phone_code":   lead.PhoneCode,
		"list_id":      lead.ListID,
	}

	if lead.PhoneCode == "" {
		params["phone_code"] = "1" // default to US
	}

	// Add optional fields only if non-empty
	optionalFields := map[string]string{
		"first_name":  lead.FirstName,
		"last_name":   lead.LastName,
		"address1":    lead.Address1,
		"city":        lead.City,
		"state":       lead.State,
		"postal_code": lead.PostalCode,
		"country":     lead.Country,
		"email":       lead.Email,
		"status":      lead.Status,
		"comments":    lead.Comments,
	}
	for k, v := range optionalFields {
		if v != "" {
			params[k] = v
		}
	}

	body, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return nil, fmt.Errorf("add_lead: %w", err)
	}

	// Response: "SUCCESS: add_lead - Lead added - 12345678"
	result := &AddLeadResult{Message: body}

	// Extract lead ID from response
	parts := strings.Split(body, " - ")
	if len(parts) >= 3 {
		// The lead ID is typically the last part
		result.LeadID = strings.TrimSpace(parts[len(parts)-1])
	}

	return result, nil
}

// UpdateLead updates an existing lead's fields.
func (c *Client) UpdateLead(ctx context.Context, leadID string,
	updates map[string]string) error {

	params := map[string]string{
		"function": "update_lead",
		"lead_id":  leadID,
	}
	for k, v := range updates {
		params[k] = v
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("update_lead: %w", err)
	}

	return nil
}

// AddLeadsBatch adds multiple leads efficiently.
// VICIdial's API doesn't have a native batch endpoint,
// so this sends concurrent requests with controlled parallelism.
func (c *Client) AddLeadsBatch(ctx context.Context, leads []Lead,
	concurrency int) ([]AddLeadResult, []error) {

	if concurrency <= 0 {
		concurrency = 5
	}

	results := make([]AddLeadResult, len(leads))
	errors := make([]error, len(leads))

	sem := make(chan struct{}, concurrency)

	type indexed struct {
		idx    int
		result *AddLeadResult
		err    error
	}
	ch := make(chan indexed, len(leads))

	for i, lead := range leads {
		sem <- struct{}{} // acquire semaphore
		go func(idx int, l Lead) {
			defer func() { <-sem }() // release semaphore
			r, err := c.AddLead(ctx, l)
			res := indexed{idx: idx, err: err}
			if r != nil {
				res.result = r
			}
			ch <- res
		}(i, lead)
	}

	// Collect results
	for range leads {
		res := <-ch
		if res.result != nil {
			results[res.idx] = *res.result
		}
		errors[res.idx] = res.err
	}

	return results, errors
}

// SearchLead searches for a lead by phone number.
func (c *Client) SearchLead(ctx context.Context,
	phoneNumber string) ([]Lead, error) {

	params := map[string]string{
		"function":     "lead_search",
		"phone_number": phoneNumber,
	}

	body, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return nil, fmt.Errorf("lead_search: %w", err)
	}

	// Parse the pipe-delimited response
	// VICIdial returns: lead_id|entry_list_id|status|user|...
	var leads []Lead
	lines := strings.Split(body, "\n")
	for _, line := range lines {
		line = strings.TrimSpace(line)
		if line == "" || strings.HasPrefix(line, "SUCCESS") {
			continue
		}
		fields := strings.Split(line, "|")
		if len(fields) >= 4 {
			leads = append(leads, Lead{
				LeadID:      fields[0],
				ListID:      fields[1],
				Status:      fields[2],
				PhoneNumber: phoneNumber,
			})
		}
	}

	return leads, nil
}

// LeadCallback schedules a callback for a lead.
func (c *Client) LeadCallback(ctx context.Context, leadID string,
	callbackDatetime string, campaignID string,
	comments string) error {

	params := map[string]string{
		"function":          "add_lead",
		"lead_id":           leadID,
		"callback":          "Y",
		"callback_datetime": callbackDatetime,
		"campaign_id":       campaignID,
		"callback_comments": comments,
		"callback_type":     "ANYONE",
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("lead_callback: %w", err)
	}

	return nil
}
```

### Usage Example: CRM Sync

```go
// cmd/crm-sync/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"time"

	vicidial "github.com/yourorg/vicidial-go"
)

type CRMContact struct {
	Phone     string `json:"phone"`
	FirstName string `json:"first_name"`
	LastName  string `json:"last_name"`
	Email     string `json:"email"`
	State     string `json:"state"`
}

func main() {
	client := vicidial.NewClient(vicidial.Config{
		BaseURL: os.Getenv("VICIDIAL_URL"),
		User:    os.Getenv("VICIDIAL_USER"),
		Pass:    os.Getenv("VICIDIAL_PASS"),
		Timeout: 30 * time.Second,
		Source:  "CRM_SYNC",
	})

	// Read contacts from CRM export
	data, err := os.ReadFile("crm_contacts.json")
	if err != nil {
		log.Fatalf("reading CRM data: %v", err)
	}

	var contacts []CRMContact
	if err := json.Unmarshal(data, &contacts); err != nil {
		log.Fatalf("parsing CRM data: %v", err)
	}

	// Convert to VICIdial leads
	leads := make([]vicidial.Lead, len(contacts))
	for i, c := range contacts {
		leads[i] = vicidial.Lead{
			PhoneNumber: c.Phone,
			FirstName:   c.FirstName,
			LastName:    c.LastName,
			Email:       c.Email,
			State:       c.State,
			ListID:      "100001", // your target list
			Status:      "NEW",
		}
	}

	// Batch import with 10 concurrent requests
	ctx := context.Background()
	results, errs := client.AddLeadsBatch(ctx, leads, 10)

	var success, failed int
	for i := range results {
		if errs[i] != nil {
			failed++
			log.Printf("FAIL lead %d (%s): %v",
				i, leads[i].PhoneNumber, errs[i])
		} else {
			success++
		}
	}

	fmt.Printf("Import complete: %d success, %d failed out of %d total\n",
		success, failed, len(leads))
}
```

---

## Real-Time Agent Monitoring

This is where Go's concurrency shines. Polling agent status every few seconds across 200 agents requires parallel requests.

```go
// monitoring.go
package vicidial

import (
	"context"
	"fmt"
	"strconv"
	"strings"
	"time"
)

// AgentStatus represents an agent's current state.
type AgentStatus struct {
	User             string
	FullName         string
	Status           string // READY, INCALL, PAUSED, DEAD, etc.
	CampaignID       string
	CallsToday       int
	PauseCode        string
	StatusSince      time.Time
	CallingLeadID    string
	PhoneNumber      string
}

// CampaignStats represents real-time campaign statistics.
type CampaignStats struct {
	CampaignID     string
	CampaignName   string
	DialLevel      float64
	AgentsLoggedIn int
	AgentsInCall   int
	AgentsWaiting  int
	AgentsPaused   int
	CallsToday     int
	DropsToday     int
	DropPercent    float64
	HopperLevel    int
	CallsInQueue   int
}

// GetAgentStatus retrieves the current status of a single agent.
func (c *Client) GetAgentStatus(ctx context.Context,
	agentUser string) (*AgentStatus, error) {

	params := map[string]string{
		"function": "agent_status",
		"agent_user": agentUser,
	}

	body, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return nil, fmt.Errorf("agent_status: %w", err)
	}

	return parseAgentStatus(body)
}

func parseAgentStatus(body string) (*AgentStatus, error) {
	// VICIdial returns pipe-delimited fields
	// Format varies by version, but typically:
	// status|campaign_id|calls_today|pause_code|...
	status := &AgentStatus{}

	// Strip the SUCCESS prefix
	body = strings.TrimPrefix(body, "SUCCESS: agent_status - ")
	fields := strings.Split(body, "|")

	if len(fields) >= 2 {
		status.Status = strings.TrimSpace(fields[0])
		status.CampaignID = strings.TrimSpace(fields[1])
	}
	if len(fields) >= 3 {
		status.CallsToday, _ = strconv.Atoi(strings.TrimSpace(fields[2]))
	}
	if len(fields) >= 4 {
		status.PauseCode = strings.TrimSpace(fields[3])
	}

	return status, nil
}

// GetCampaignStats retrieves real-time statistics for a campaign.
func (c *Client) GetCampaignStats(ctx context.Context,
	campaignID string) (*CampaignStats, error) {

	params := map[string]string{
		"function":    "campaign_stats",
		"campaign_id": campaignID,
	}

	body, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return nil, fmt.Errorf("campaign_stats: %w", err)
	}

	return parseCampaignStats(body, campaignID)
}

func parseCampaignStats(body string, campaignID string) (*CampaignStats, error) {
	stats := &CampaignStats{
		CampaignID: campaignID,
	}

	body = strings.TrimPrefix(body, "SUCCESS: campaign_stats - ")

	// Parse key-value pairs from the response
	// VICIdial returns something like:
	// dial_level: 3.0|agents_logged_in: 45|...
	pairs := strings.Split(body, "|")
	for _, pair := range pairs {
		kv := strings.SplitN(pair, ":", 2)
		if len(kv) != 2 {
			continue
		}
		key := strings.TrimSpace(kv[0])
		val := strings.TrimSpace(kv[1])

		switch key {
		case "dial_level":
			stats.DialLevel, _ = strconv.ParseFloat(val, 64)
		case "agents_logged_in":
			stats.AgentsLoggedIn, _ = strconv.Atoi(val)
		case "agents_incall":
			stats.AgentsInCall, _ = strconv.Atoi(val)
		case "agents_waiting":
			stats.AgentsWaiting, _ = strconv.Atoi(val)
		case "agents_paused":
			stats.AgentsPaused, _ = strconv.Atoi(val)
		case "calls_today":
			stats.CallsToday, _ = strconv.Atoi(val)
		case "drops_today":
			stats.DropsToday, _ = strconv.Atoi(val)
		case "drop_percent":
			stats.DropPercent, _ = strconv.ParseFloat(val, 64)
		case "hopper_level":
			stats.HopperLevel, _ = strconv.Atoi(val)
		case "calls_in_queue":
			stats.CallsInQueue, _ = strconv.Atoi(val)
		}
	}

	return stats, nil
}

// MonitorConfig controls the behavior of the real-time monitor.
type MonitorConfig struct {
	// CampaignIDs to monitor
	CampaignIDs []string

	// PollInterval is how often to fetch data (default: 5s)
	PollInterval time.Duration

	// OnUpdate is called with fresh stats on each poll
	OnUpdate func(stats []CampaignStats)

	// OnError is called when a poll fails
	OnError func(campaignID string, err error)
}

// Monitor polls campaign stats at regular intervals.
// It runs until the context is cancelled.
func (c *Client) Monitor(ctx context.Context, cfg MonitorConfig) error {
	interval := cfg.PollInterval
	if interval == 0 {
		interval = 5 * time.Second
	}

	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			stats := make([]CampaignStats, 0, len(cfg.CampaignIDs))
			ch := make(chan struct {
				stats *CampaignStats
				err   error
				id    string
			}, len(cfg.CampaignIDs))

			// Poll all campaigns concurrently
			for _, id := range cfg.CampaignIDs {
				go func(campaignID string) {
					s, err := c.GetCampaignStats(ctx, campaignID)
					ch <- struct {
						stats *CampaignStats
						err   error
						id    string
					}{s, err, campaignID}
				}(id)
			}

			// Collect results
			for range cfg.CampaignIDs {
				result := <-ch
				if result.err != nil {
					if cfg.OnError != nil {
						cfg.OnError(result.id, result.err)
					}
				} else if result.stats != nil {
					stats = append(stats, *result.stats)
				}
			}

			if cfg.OnUpdate != nil && len(stats) > 0 {
				cfg.OnUpdate(stats)
			}
		}
	}
}
```

### Usage Example: Terminal Dashboard

```go
// cmd/dashboard/main.go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"strings"
	"time"

	vicidial "github.com/yourorg/vicidial-go"
)

func main() {
	client := vicidial.NewClient(vicidial.Config{
		BaseURL: os.Getenv("VICIDIAL_URL"),
		User:    os.Getenv("VICIDIAL_USER"),
		Pass:    os.Getenv("VICIDIAL_PASS"),
		Source:  "DASHBOARD",
	})

	campaigns := strings.Split(os.Getenv("CAMPAIGNS"), ",")
	if len(campaigns) == 0 || campaigns[0] == "" {
		log.Fatal("Set CAMPAIGNS env var (comma-separated campaign IDs)")
	}

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
	defer cancel()

	fmt.Println("VICIdial Real-Time Dashboard")
	fmt.Println("Press Ctrl+C to exit")
	fmt.Println()

	err := client.Monitor(ctx, vicidial.MonitorConfig{
		CampaignIDs:  campaigns,
		PollInterval: 5 * time.Second,
		OnUpdate: func(stats []vicidial.CampaignStats) {
			// Clear screen
			fmt.Print("\033[2J\033[H")
			fmt.Printf("VICIdial Dashboard — %s\n\n",
				time.Now().Format("15:04:05"))

			fmt.Printf("%-12s %6s %6s %6s %6s %8s %6s %7s %6s\n",
				"Campaign", "Agents", "InCall", "Wait", "Pause",
				"Calls", "Drops", "Drop%", "Hopper")
			fmt.Println(strings.Repeat("-", 78))

			var totalAgents, totalCalls, totalDrops int
			for _, s := range stats {
				fmt.Printf("%-12s %6d %6d %6d %6d %8d %6d %6.1f%% %6d\n",
					s.CampaignID,
					s.AgentsLoggedIn,
					s.AgentsInCall,
					s.AgentsWaiting,
					s.AgentsPaused,
					s.CallsToday,
					s.DropsToday,
					s.DropPercent,
					s.HopperLevel,
				)
				totalAgents += s.AgentsLoggedIn
				totalCalls += s.CallsToday
				totalDrops += s.DropsToday
			}

			fmt.Println(strings.Repeat("-", 78))
			overallDrop := 0.0
			if totalCalls > 0 {
				overallDrop = float64(totalDrops) / float64(totalCalls) * 100
			}
			fmt.Printf("%-12s %6d %6s %6s %6s %8d %6d %6.1f%%\n",
				"TOTAL", totalAgents, "", "", "",
				totalCalls, totalDrops, overallDrop)
		},
		OnError: func(id string, err error) {
			log.Printf("Error polling %s: %v", id, err)
		},
	})

	if err != nil && err != context.Canceled {
		log.Fatalf("Monitor error: %v", err)
	}
}
```

Running this gives you a live-updating terminal dashboard:

```
VICIdial Dashboard — 14:23:47

Campaign     Agents InCall   Wait  Pause    Calls  Drops   Drop% Hopper
------------------------------------------------------------------------------
SALES_B2B        45     32      8      5     4823    127    2.6%   2847
SOLAR_OB         28     19      6      3     2941     82    2.8%   1523
INSURANCE        67     48     12      7     7294    198    2.7%   4291
------------------------------------------------------------------------------
TOTAL           140                         15058    407    2.7%
```

---

## Campaign Control

Operations that change campaign behavior — adjusting dial levels, pausing agents, managing callbacks.

```go
// campaigns.go
package vicidial

import (
	"context"
	"fmt"
)

// SetDialLevel changes the auto-dial level for a campaign.
func (c *Client) SetDialLevel(ctx context.Context,
	campaignID string, level float64) error {

	params := map[string]string{
		"function":    "update_campaign",
		"campaign_id": campaignID,
		"dial_level":  fmt.Sprintf("%.1f", level),
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("set_dial_level: %w", err)
	}

	return nil
}

// PauseAgent puts an agent into PAUSED state.
func (c *Client) PauseAgent(ctx context.Context,
	agentUser string, pauseCode string) error {

	params := map[string]string{
		"function":   "pause_agent",
		"agent_user": agentUser,
	}
	if pauseCode != "" {
		params["value"] = pauseCode
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("pause_agent: %w", err)
	}

	return nil
}

// ResumeAgent takes an agent out of PAUSED state.
func (c *Client) ResumeAgent(ctx context.Context,
	agentUser string) error {

	params := map[string]string{
		"function":   "resume_agent",
		"agent_user": agentUser,
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("resume_agent: %w", err)
	}

	return nil
}

// LogoutAgent forces an agent logout.
func (c *Client) LogoutAgent(ctx context.Context,
	agentUser string) error {

	params := map[string]string{
		"function":   "logout_agent",
		"agent_user": agentUser,
	}

	_, err := c.nonAgentAPI(ctx, params)
	if err != nil {
		return fmt.Errorf("logout_agent: %w", err)
	}

	return nil
}

// DropRateGuard monitors drop rate and automatically reduces
// the dial level when it exceeds the threshold.
func (c *Client) DropRateGuard(ctx context.Context,
	campaignID string, maxDropRate float64,
	checkInterval int) error {

	// This is a simplified version. In production, you'd want
	// hysteresis (don't immediately crank the level back up),
	// a minimum dial level, and notification when it triggers.

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		stats, err := c.GetCampaignStats(ctx, campaignID)
		if err != nil {
			fmt.Printf("DropRateGuard: error getting stats: %v\n", err)
			continue
		}

		if stats.DropPercent > maxDropRate && stats.DialLevel > 1.0 {
			newLevel := stats.DialLevel - 0.5
			if newLevel < 1.0 {
				newLevel = 1.0
			}
			fmt.Printf("DropRateGuard: %s drop rate %.1f%% exceeds %.1f%%, "+
				"reducing dial level from %.1f to %.1f\n",
				campaignID, stats.DropPercent, maxDropRate,
				stats.DialLevel, newLevel)

			if err := c.SetDialLevel(ctx, campaignID, newLevel); err != nil {
				fmt.Printf("DropRateGuard: error setting dial level: %v\n", err)
			}
		}
	}
}
```

### Usage Example: Drop Rate Protection

```go
// cmd/dropguard/main.go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"

	vicidial "github.com/yourorg/vicidial-go"
)

func main() {
	client := vicidial.NewClient(vicidial.Config{
		BaseURL: os.Getenv("VICIDIAL_URL"),
		User:    os.Getenv("VICIDIAL_USER"),
		Pass:    os.Getenv("VICIDIAL_PASS"),
		Source:  "DROP_GUARD",
	})

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
	defer cancel()

	// Monitor SALES campaign, reduce dial level if drops exceed 2.5%
	// (below the 3% FTC/TCPA threshold to give buffer)
	log.Println("Starting drop rate guard for SALES campaign (max 2.5%)")
	err := client.DropRateGuard(ctx, "SALES", 2.5, 30)
	if err != nil && err != context.Canceled {
		log.Fatalf("DropRateGuard error: %v", err)
	}
}
```

This is the kind of automation that prevents TCPA violations. The dial level automatically ratchets down when drop rates approach the 3% limit, without requiring a human to watch the real-time report.

---

## Testing

Testing VICIdial API integrations requires a mock server because you don't want your tests hitting a production dialer.

```go
// client_test.go
package vicidial

import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestAddLead_Success(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
			// Verify required parameters
			q := r.URL.Query()
			if q.Get("function") != "add_lead" {
				t.Errorf("expected function=add_lead, got %s",
					q.Get("function"))
			}
			if q.Get("phone_number") != "5551234567" {
				t.Errorf("expected phone=5551234567, got %s",
					q.Get("phone_number"))
			}
			if q.Get("user") == "" || q.Get("pass") == "" {
				t.Error("missing credentials")
			}

			w.WriteHeader(200)
			w.Write([]byte("SUCCESS: add_lead - Lead added - 99887766"))
		},
	))
	defer server.Close()

	client := NewClient(Config{
		BaseURL: server.URL,
		User:    "testuser",
		Pass:    "testpass",
	})

	result, err := client.AddLead(context.Background(), Lead{
		PhoneNumber: "5551234567",
		ListID:      "100",
		FirstName:   "John",
		LastName:    "Doe",
	})

	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if result.LeadID != "99887766" {
		t.Errorf("expected leadID 99887766, got %s", result.LeadID)
	}
}

func TestAddLead_APIError(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(200) // VICIdial returns 200 even on errors
			w.Write([]byte(
				"ERROR: add_lead - DUPLICATE PHONE NUMBER IN LIST"))
		},
	))
	defer server.Close()

	client := NewClient(Config{
		BaseURL: server.URL,
		User:    "testuser",
		Pass:    "testpass",
	})

	_, err := client.AddLead(context.Background(), Lead{
		PhoneNumber: "5551234567",
		ListID:      "100",
	})

	if err == nil {
		t.Fatal("expected error, got nil")
	}

	apiErr, ok := err.(*APIError)
	if !ok {
		// Unwrap fmt.Errorf wrapper
		t.Logf("error type: %T, message: %v", err, err)
	} else {
		if apiErr.Function != "add_lead" {
			t.Errorf("expected function add_lead, got %s", apiErr.Function)
		}
	}
}

func TestSearchLead(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(200)
			w.Write([]byte(
				"SUCCESS: lead_search - results found\n" +
				"12345|100|NEW|agent1\n" +
				"12346|101|CALLBK|agent2\n"))
		},
	))
	defer server.Close()

	client := NewClient(Config{
		BaseURL: server.URL,
		User:    "testuser",
		Pass:    "testpass",
	})

	leads, err := client.SearchLead(context.Background(), "5551234567")
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	if len(leads) != 2 {
		t.Fatalf("expected 2 leads, got %d", len(leads))
	}
	if leads[0].LeadID != "12345" {
		t.Errorf("expected lead ID 12345, got %s", leads[0].LeadID)
	}
	if leads[1].Status != "CALLBK" {
		t.Errorf("expected status CALLBK, got %s", leads[1].Status)
	}
}

func TestCampaignStats(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(200)
			w.Write([]byte(
				"SUCCESS: campaign_stats - " +
				"dial_level: 3.0|agents_logged_in: 45|agents_incall: 32|" +
				"agents_waiting: 8|agents_paused: 5|calls_today: 4823|" +
				"drops_today: 127|drop_percent: 2.6|hopper_level: 2847"))
		},
	))
	defer server.Close()

	client := NewClient(Config{
		BaseURL: server.URL,
		User:    "testuser",
		Pass:    "testpass",
	})

	stats, err := client.GetCampaignStats(context.Background(), "SALES")
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	if stats.AgentsLoggedIn != 45 {
		t.Errorf("expected 45 agents, got %d", stats.AgentsLoggedIn)
	}
	if stats.DialLevel != 3.0 {
		t.Errorf("expected dial level 3.0, got %.1f", stats.DialLevel)
	}
	if stats.DropPercent != 2.6 {
		t.Errorf("expected drop 2.6%%, got %.1f%%", stats.DropPercent)
	}
}
```

Run the tests:

```bash
go test -v ./...
```

---

## Deployment Patterns

### Pattern 1: CLI Tool

Build a CLI that your ops team uses for ad-hoc operations:

```go
// cmd/vicli/main.go
package main

import (
	"context"
	"fmt"
	"os"
	"time"

	vicidial "github.com/yourorg/vicidial-go"
)

func main() {
	client := vicidial.NewClient(vicidial.Config{
		BaseURL: os.Getenv("VICIDIAL_URL"),
		User:    os.Getenv("VICIDIAL_USER"),
		Pass:    os.Getenv("VICIDIAL_PASS"),
		Timeout: 15 * time.Second,
	})

	if len(os.Args) < 2 {
		fmt.Println("Usage: vicli <command> [args]")
		fmt.Println("Commands:")
		fmt.Println("  stats <campaign_id>    Campaign statistics")
		fmt.Println("  search <phone>         Search for a lead")
		fmt.Println("  dial-level <campaign> <level>  Set dial level")
		fmt.Println("  pause <agent_user> [code]      Pause agent")
		fmt.Println("  resume <agent_user>            Resume agent")
		os.Exit(1)
	}

	ctx := context.Background()

	switch os.Args[1] {
	case "stats":
		if len(os.Args) < 3 {
			fmt.Println("Usage: vicli stats <campaign_id>")
			os.Exit(1)
		}
		stats, err := client.GetCampaignStats(ctx, os.Args[2])
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error: %v\n", err)
			os.Exit(1)
		}
		fmt.Printf("Campaign: %s\n", stats.CampaignID)
		fmt.Printf("Dial Level: %.1f\n", stats.DialLevel)
		fmt.Printf("Agents: %d logged in, %d in call, %d waiting, %d paused\n",
			stats.AgentsLoggedIn, stats.AgentsInCall,
			stats.AgentsWaiting, stats.AgentsPaused)
		fmt.Printf("Calls Today: %d (drops: %d, %.1f%%)\n",
			stats.CallsToday, stats.DropsToday, stats.DropPercent)
		fmt.Printf("Hopper: %d leads\n", stats.HopperLevel)

	case "search":
		if len(os.Args) < 3 {
			fmt.Println("Usage: vicli search <phone_number>")
			os.Exit(1)
		}
		leads, err := client.SearchLead(ctx, os.Args[2])
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error: %v\n", err)
			os.Exit(1)
		}
		if len(leads) == 0 {
			fmt.Println("No leads found")
		}
		for _, l := range leads {
			fmt.Printf("Lead %s | List %s | Status: %s\n",
				l.LeadID, l.ListID, l.Status)
		}

	default:
		fmt.Fprintf(os.Stderr, "Unknown command: %s\n", os.Args[1])
		os.Exit(1)
	}
}
```

Build and deploy:

```bash
# Build for Linux (your VICIdial server)
GOOS=linux GOARCH=amd64 go build -o vicli ./cmd/vicli/

# Copy to server
scp vicli root@dialer:/usr/local/bin/

# Use it
ssh root@dialer 'VICIDIAL_URL=https://localhost VICIDIAL_USER=api VICIDIAL_PASS=secret vicli stats SALES'
```

### Pattern 2: HTTP Middleware

Expose VICIdial operations as a REST API with proper JSON responses, auth tokens, and rate limiting. This is useful when you need to integrate VICIdial with web applications that expect modern APIs.

```go
// cmd/api-proxy/main.go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"

	vicidial "github.com/yourorg/vicidial-go"
)

var client *vicidial.Client

func main() {
	client = vicidial.NewClient(vicidial.Config{
		BaseURL: os.Getenv("VICIDIAL_URL"),
		User:    os.Getenv("VICIDIAL_USER"),
		Pass:    os.Getenv("VICIDIAL_PASS"),
		Source:  "API_PROXY",
	})

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/campaigns/{id}/stats", handleCampaignStats)
	mux.HandleFunc("POST /api/leads", handleAddLead)
	mux.HandleFunc("GET /api/leads/search", handleSearchLead)

	server := &http.Server{
		Addr:         ":8080",
		Handler:      authMiddleware(mux),
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	log.Println("VICIdial API proxy listening on :8080")
	log.Fatal(server.ListenAndServe())
}

func authMiddleware(next http.Handler) http.Handler {
	apiKey := os.Getenv("API_KEY")
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer "+apiKey {
			http.Error(w, `{"error":"unauthorized"}`, 401)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func handleCampaignStats(w http.ResponseWriter, r *http.Request) {
	campaignID := r.PathValue("id")
	stats, err := client.GetCampaignStats(r.Context(), campaignID)
	if err != nil {
		writeJSON(w, 500, map[string]string{"error": err.Error()})
		return
	}
	writeJSON(w, 200, stats)
}

func handleAddLead(w http.ResponseWriter, r *http.Request) {
	var lead vicidial.Lead
	if err := json.NewDecoder(r.Body).Decode(&lead); err != nil {
		writeJSON(w, 400, map[string]string{"error": "invalid request body"})
		return
	}

	result, err := client.AddLead(r.Context(), lead)
	if err != nil {
		writeJSON(w, 500, map[string]string{"error": err.Error()})
		return
	}
	writeJSON(w, 201, result)
}

func handleSearchLead(w http.ResponseWriter, r *http.Request) {
	phone := r.URL.Query().Get("phone")
	if phone == "" {
		writeJSON(w, 400, map[string]string{"error": "phone parameter required"})
		return
	}

	leads, err := client.SearchLead(context.Background(), phone)
	if err != nil {
		writeJSON(w, 500, map[string]string{"error": err.Error()})
		return
	}
	writeJSON(w, 200, map[string]interface{}{"leads": leads})
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}
```

Now your web team can integrate with VICIdial using normal REST calls:

```bash
# Get campaign stats
curl -H "Authorization: Bearer your-api-key" \
  http://localhost:8080/api/campaigns/SALES/stats

# Add a lead
curl -X POST -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"phone_number":"5551234567","first_name":"Jane","last_name":"Smith","list_id":"100"}' \
  http://localhost:8080/api/leads

# Search by phone
curl -H "Authorization: Bearer your-api-key" \
  "http://localhost:8080/api/leads/search?phone=5551234567"
```

---

## What We Didn't Cover (Yet)

This library covers the most common operations. VICIdial's API has many more functions:

- **Recording access** — download call recordings by lead ID or date range
- **DNC list management** — add/remove numbers from internal DNC lists
- **List upload** — bulk lead import via file upload (different from the per-lead API)
- **Agent screen control** — send URLs to agent screens, transfer calls, conference calls
- **Custom fields** — VICIdial supports custom lead fields that map to additional columns

The pattern for adding any of these is the same: build a method on the client struct, pass the right parameters, parse the response. Once you've seen the pattern three times, adding a new API function takes about 10 minutes.

The full library is on our GitHub. If you'd rather skip the API work entirely and just get VICIdial doing what you need, [ViciStack's managed service](https://vicistack.com) includes all integrations and monitoring out of the box.

---

## Resources

- [VICIdial API Integration Guide](https://vicistack.com/blog/vicidial-api-integration/) — complete API reference
- [VICIdial CRM Integration](https://vicistack.com/blog/vicidial-crm-integration/) — CRM-specific integration patterns
- [VICIdial Non-Agent API Documentation](https://vicidial.org/docs/NON-AGENT_API.txt) — official API reference

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-golang-api-client).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
