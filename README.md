# StockMaster
Let’s combine the two concepts into a cohesive 1-week project: a **High-Performance Stock Trading Dashboard in React Native with a Chatbot**, backed by a Go + Gin server. The dashboard will integrate real-time public market data and native libraries for performance, while the chatbot adds interactivity (e.g., answering stock queries). Below is the updated plan, integrating all key discussion points, with a 7-day timeline and learning resources.

---

### Project Overview
- **Frontend**: React Native app with a stock dashboard (charts, watchlist, ticker) and chatbot UI.
- **Backend**: Go + Gin server for API proxying, WebSocket streaming, and chatbot logic.
- **Chatbot**: Rule-based bot for stock queries (e.g., “What’s AAPL’s price?”).
- **Performance Goals**: Leverage native libraries, caching, and optimized rendering.
- **Timeline**: 1 week (7 days, ~8 hours/day).

---

### Revised Plan: Dashboard + Chatbot in 1 Week

#### Day 1: Setup & Backend Foundation
- **Objective**: Set up frontend and backend, proxy Finnhub API.
- **Frontend**: Initialize React Native (`npx react-native init StockDashboard`).
- **Backend**: Set up Go + Gin.
  - Install Go: `brew install go` (Mac) or download from [golang.org](https://golang.org).
  - Add Gin: `go get -u github.com/gin-gonic/gin`.
  - Create `/health` and `/api/stock/:symbol` endpoints.
- **Security**: Hide Finnhub API key in backend (use `.env` with `github.com/joho/godotenv`).
- **Deliverable**: Backend proxies Finnhub data (e.g., `GET /api/stock/AAPL`).

**Code (main.go)**:
```go
package main

import (
	"net/http"
	"os"
	"github.com/gin-gonic/gin"
	"github.com/joho/godotenv"
)

func main() {
	godotenv.Load()
	r := gin.Default()

	r.GET("/health", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	})

	r.GET("/api/stock/:symbol", func(c *gin.Context) {
		symbol := c.Param("symbol")
		apiKey := os.Getenv("FINNHUB_API_KEY")
		url := "https://finnhub.io/api/v1/quote?symbol=" + symbol + "&token=" + apiKey
		resp, _ := http.Get(url)
		defer resp.Body.Close()
		c.Data(http.StatusOK, "application/json", resp.Body)
	})

	r.Run(":8080")
}
```

**Learning Resources**:
- [Go Basics](https://go.dev/tour/welcome/1)
- [Gin Docs](https://gin-gonic.com/docs/)
- [Finnhub API](https://finnhub.io/docs/api)

---

#### Day 2: Real-Time Data with WebSockets
- **Objective**: Add real-time updates via WebSockets.
- **Backend**: Use `github.com/gorilla/websocket` for `/ws/stock/:symbol`.
  - Connect to Finnhub WebSocket: `wss://ws.finnhub.io?token=<API_KEY>`.
- **Frontend**: Connect to backend WebSocket with native `WebSocket` API.
- **Data Handling**: Test with AAPL updates.
- **Deliverable**: Real-time AAPL price in the app.

**Backend (WebSocket)**:
```go
import (
	"github.com/gorilla/websocket"
	"time"
)

var upgrader = websocket.Upgrader{}

func handleWebSocket(c *gin.Context) {
	symbol := c.Param("symbol")
	conn, _ := upgrader.Upgrade(c.Writer, c.Request, nil)
	defer conn.Close()

	// Simulate real-time data (replace with Finnhub WebSocket)
	for {
		conn.WriteJSON(gin.H{"symbol": symbol, "price": 150.00 + float64(rand.Intn(10))})
		time.Sleep(1 * time.Second)
	}
}

r.GET("/ws/stock/:symbol", handleWebSocket)
```

**Frontend (App.js snippet)**:
```jsx
const [price, setPrice] = useState(0);

useEffect(() => {
  const ws = new WebSocket('ws://localhost:8080/ws/stock/AAPL');
  ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
  return () => ws.close();
}, []);

return <Text>AAPL: ${price}</Text>;
```

**Learning Resources**:
- [WebSocket in Go](https://github.com/gorilla/websocket)
- [Finnhub WebSocket](https://finnhub.io/docs/api/websocket)
- [React Native WebSocket](https://reactnative.dev/docs/network#websocket-support)

---

#### Day 3: Dashboard UI & Charting
- **Objective**: Build dashboard with charts and watchlist.
- **Frontend**:
  - **Watchlist**: FlatList with dummy data.
  - **Chart**: Use `react-native-charts-wrapper` (install: `npm i react-native-charts-wrapper`).
  - **Ticker**: Text component with WebSocket data.
- **Deliverable**: Dashboard with AAPL chart and ticker.

**Code (App.js)**:
```jsx
import { FlatList, Text, View } from 'react-native';
import { CandlestickChart } from 'react-native-charts-wrapper';

export default function App() {
  const [price, setPrice] = useState(0);
  const data = { dataSets: [{ values: [{ x: 1, shadowH: 152, shadowL: 148, open: 150, close: price }] }] };

  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080/ws/stock/AAPL');
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
    return () => ws.close();
  }, []);

  return (
    <View>
      <Text>Ticker: AAPL ${price}</Text>
      <CandlestickChart style={{ height: 200 }} data={data} />
      <FlatList data={[{ key: 'AAPL' }]} renderItem={({ item }) => <Text>{item.key}</Text>} />
    </View>
  );
}
```

**Learning Resources**:
- [React Native Docs](https://reactnative.dev/docs/components-and-apis)
- [Charts Wrapper](https://github.com/wuxudong/react-native-charts-wrapper)

---

#### Day 4: Chatbot Backend
- **Objective**: Add chatbot logic in Go.
- **Backend**: `/chat` endpoint with rule-based parsing.
- **Deliverable**: Chatbot returns stock prices.

**Code (main.go)**:
```go
import "strings"

func handleChat(c *gin.Context) {
	var req struct { Message string `json:"message"` }
	c.BindJSON(&req)

	msg := strings.ToLower(req.Message)
	if strings.Contains(msg, "price of") {
		symbol := strings.TrimSpace(strings.Split(msg, "price of")[1])
		url := "https://finnhub.io/api/v1/quote?symbol=" + symbol + "&token=" + os.Getenv("FINNHUB_API_KEY")
		resp, _ := http.Get(url)
		defer resp.Body.Close()
		var data map[string]interface{}
		json.NewDecoder(resp.Body).Decode(&data)
		price := data["c"].(float64)
		c.JSON(http.StatusOK, gin.H{"response": fmt.Sprintf("%s price is $%.2f", symbol, price)})
		return
	}
	c.JSON(http.StatusOK, gin.H{"response": "Sorry, I don’t understand."})
}

r.POST("/chat", handleChat)
```

**Learning Resources**:
- [Go Strings](https://pkg.go.dev/strings)
- [JSON in Go](https://go.dev/blog/json)

---

#### Day 5: Chatbot Frontend & Optimization
- **Objective**: Integrate chatbot UI, optimize performance.
- **Frontend**: Add chat UI, use `react-query` for caching API calls.
- **Performance**: Set `initialNumToRender` on FlatList.
- **Deliverable**: Chatbot works in app.

**Code (App.js snippet)**:
```jsx
const [messages, setMessages] = useState([]);
const [input, setInput] = useState('');

const sendMessage = async () => {
  const res = await fetch('http://localhost:8080/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: input }),
  });
  const { response } = await res.json();
  setMessages([...messages, { text: input, from: 'user' }, { text: response, from: 'bot' }]);
  setInput('');
};

return (
  <View>
    <FlatList data={messages} renderItem={({ item }) => <Text>{item.from}: {item.text}</Text>} initialNumToRender={10} />
    <TextInput value={input} onChangeText={setInput} />
    <Button title="Send" onPress={sendMessage} />
  </View>
);
```

**Learning Resources**:
- [React Query](https://tanstack.com/query/v4/docs/react-native)
- [FlatList Optimization](https://reactnative.dev/docs/optimizing-flatlist-configuration)

---

#### Day 6: Native Libraries & Polish
- **Objective**: Experiment with native charting, polish UI.
- **Frontend**: Test `react-native-charts-wrapper` vs. a Native Module (e.g., MPAndroidChart).
  - Basic Native Module guide: [React Native Docs](https://reactnative.dev/docs/native-modules-ios).
- **Backend**: Add middleware for API key security.
- **Deliverable**: Optimized dashboard with secure backend.

**Gin Middleware**:
```go
func authMiddleware(c *gin.Context) {
	if c.GetHeader("X-API-Key") != "your-secret-key" {
		c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
		return
	}
	c.Next()
}

r.Use(authMiddleware)
```

**Learning Resources**:
- [Native Modules](https://reactnative.dev/docs/native-modules-android)
- [MPAndroidChart](https://github.com/PhilJay/MPAndroidChart)

---

#### Day 7: Testing & Deployment
- **Objective**: Test and deploy.
- **Testing**: Run on device (`react-native run-android/ios`).
- **Deployment**: Deploy backend to Render.com, update frontend URL.
- **Deliverable**: Fully functional app.

**Learning Resources**:
- [Render Deployment](https://render.com/docs/deploy-go)
- [React Native Debugging](https://reactnative.dev/docs/debugging)

---

### Final Architecture
- **Frontend**: React Native with WebSocket, REST, `react-native-charts-wrapper`, and chatbot UI.
- **Backend**: Go + Gin with API proxy, WebSocket streaming, and chatbot logic.
- **Data**: Finnhub API, proxied and streamed.

---

### Learning Sources Summary
1. **React Native**: [Official Docs](https://reactnative.dev/docs/getting-started)
2. **Go**: [Tour of Go](https://go.dev/tour/welcome/1), [Effective Go](https://go.dev/doc/effective_go)
3. **Gin**: [Gin Docs](https://gin-gonic.com/docs/)
4. **Finnhub**: [API Docs](https://finnhub.io/docs/api)
5. **WebSockets**: [Gor [Gorilla WebSocket](https://github.com/gorilla/websocket)
6. **Charting**: [react-native-charts-wrapper](https://github.com/wuxudong/react-native-charts-wrapper)
7. **Performance**: [React Query](https://tanstack.com/query/v4/docs/react-native), [Reanimated](https://docs.swmansion.com/react-native-reanimated/)

---

### Tips
- **Focus**: Prioritize WebSocket and chatbot over native modules if time runs short.
- **Debug**: Use `react-native-debugger` and Go’s `log`.

Ready to start Day 1? Let me know if you want more code or setup help!
