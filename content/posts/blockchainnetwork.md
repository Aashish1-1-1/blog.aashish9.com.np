---
title: "How does a blockchain network work?"
date: 2025-08-15
draft: false
---

## Back Story

Hey, Are you curious? Curious enough to question things around you, Curious enough to dig deeper to it's core, Curious enough to learn more than other, Curious enough to break things and join them back.

My answer is I don't know, sometime I am and sometime I'm not and It is a story when I was one. I was looking to white paper of bitcoin and learning how is the network self sustaining and reliable in itself. I was chatting with llm to know about it more what are the types of blockchain network architecture. This post is just about explain one of the architecture called mining pool setup.

## Introduction

In the mentioned architecture Blockchain network is divided in basically 4 parts and they are:

- Seed Server
- Full Node
- Miners
- User And Wallet

Below is the the diagram representation of it (please toggle theme to see the edges sorry for inconvenience)
![concept](/images/nodes.png)

Now let me explain each parts

- Seed Server:
  A seed server (sometimes called a DNS seed) is basically a special server that helps a new node find other peers when it first joins the network.So it keep track of node that may be active.

- Full Node:
  Full node stores entire blockchain,it also maintain mempool of transactions that are to be mined.It is main component to participate in peertopeer networking,connects multiple peers shares tranactions broadcast if hash sent by miner.

- Miners
  Miner task is to find hash for transactions. Multiple miners compete for finding hash first after finding hash they broadcast it to connected nearby full node.

- User and Wallet
  It is a component to initiate transaction.

## Working

Let's talk about how these components works together:

- When a user initiate a transaction eg: A send 2btc to B,Wallet send that transaction to full node through http/websocket or tcp if wallet is also a node
- Now full node receives that transaction it validates it like if user A hash that btc or not and prepares transaction, it appends transaction to mempool and Gossips to all its' peers. Peers can be miners or other full nodes.
- If the receiving node is miner they go for finding the hash according to the difficulty of blockchain else if it is other full node it also appends transaction to its full node and regossips it to connected node this gossip is done in TCP.
- Let's assume miner1 found hash it immediately broadcast that hash to the connected near full node and full node rebroadcast to other connected nodes.
- Now on receiving hash full node verifies the hash.As it is **Hard to find,easy to verify**. Full node then appends that block with hash to rewards the miner with fee.

## Implementaion

I tried to implement this flow of every components as close I could get. Its' not anywhere close to perfection but I mean I tried my best. Golang is used for implemention. Every component is contenerized and every container have same externall network called blockchain-network so that each components can communicate with each other.More technical details of it can be seen through below repo link

Seed server

```golang
func NewReq(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	defer r.Body.Close()

	body, err := io.ReadAll(r.Body)
	if err != nil {
		fmt.Printf("Error reading request body: %v\n", err)
		http.Error(w, "Failed to read request body", http.StatusInternalServerError)
		return
	}

	if len(body) == 0 {
		http.Error(w, "Empty request body", http.StatusBadRequest)
		return
	}

	var newNode set.Nodes
	if err := json.Unmarshal(body, &newNode); err != nil {
		fmt.Printf("Error unmarshaling JSON: %v\n", err)
		http.Error(w, "Invalid JSON format", http.StatusBadRequest)
		return
	}

	fmt.Printf("Received registration request from node: %s %s\n", newNode.Ip, newNode.HTTP_PORT)

	mu.Lock()
	tracknodes.Add(newNode.Ip, newNode)
	mu.Unlock()

	fmt.Printf("Added node %s to tracker. Total nodes: %d\n", newNode.Ip, len(tracknodes))

	peerNode, found := getNextAvailableNode(newNode)

	response := map[string]interface{}{}

	if !found {
		response["message"] = set.Nodes{}
		response["status"] = "first_node"
		fmt.Printf("No peers available for %s - first node in network\n", newNode.Ip)
	} else {
		response["message"] = peerNode
		response["status"] = "peer_assigned"
		fmt.Printf("Assigned peer %s to new node %s\n", peerNode.Ip, newNode.Ip)
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		fmt.Printf("Error encoding response: %v\n", err)
	}
}
func checker() {
	fmt.Println("Node checker service started")

	for {
		for len(tracknodes) == 0 {
			fmt.Println("Checker: Waiting for nodes...")
			time.Sleep(5 * time.Second)
		}

		fmt.Printf("Checker: Pinging %d nodes\n", len(tracknodes))

		// Create a slice to store nodes to remove (to avoid modifying map while iterating)
		var nodesToRemove []string

		mu.RLock()
		for ip, node := range tracknodes {
			url := ip + ":" + node.HTTP_PORT
			fmt.Println(url)
			resp, err := http.Get(url)
			if err != nil {
				fmt.Printf("Checker: Node %s is unreachable: %v\n", url, err)
				nodesToRemove = append(nodesToRemove, ip)
				continue
			}

			resp.Body.Close()

			if resp.StatusCode != http.StatusOK {
				fmt.Printf("Checker: Node %s returned status %d\n", ip, resp.StatusCode)
				nodesToRemove = append(nodesToRemove, ip)
			} else {
				fmt.Printf("Checker: Node %s is healthy\n", ip)
			}
		}
		mu.RUnlock()

		if len(nodesToRemove) > 0 {
			mu.Lock()
			for _, ip := range nodesToRemove {
				tracknodes.Remove(ip)
				fmt.Printf("Removed unhealthy node: %s\n", ip)
			}
			mu.Unlock()
		}

		time.Sleep(30 * time.Second)
	}
}
func main() {
	fmt.Println("Starting Seed Server...")

	go checker()

	// Setup routes
	http.Handle("/", CORS(NewReq))
	http.Handle("/sendonly", CORS(SendOnly))

	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"status":     "healthy",
			"node_count": len(tracknodes),
		})
	})

	fmt.Println("Seed Server started on :8080")
	fmt.Println("Endpoints:")
	fmt.Println("  POST /        - Register new node")
	fmt.Println("  GET  /sendonly - Get random node")
	fmt.Println("  GET  /health  - Health check")

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Tcp header handle

```golang
func (peers PeersMap) HandleClient(conn net.Conn) {
	defer func() {
		fmt.Printf("Miner: Connection to %s closed\n", conn.RemoteAddr())
		conn.Close()

		// Remove from peers map when connection closes
		for peerKey, peerConn := range peers {
			if peerConn != nil && peerConn.Conn == conn {
				delete(peers, peerKey)
				fmt.Printf("Miner: Removed peer %s from peers map\n", peerKey)
				break
			}
		}
	}()

	fmt.Printf("Miner: Started handling messages from %s\n", conn.RemoteAddr())

	for {
		header, err := readFull(conn, 8)
		if err != nil {
			if err == io.EOF {
				fmt.Printf("Miner: Connection to %s closed by remote\n", conn.RemoteAddr())
			} else {
				fmt.Printf("Miner: Error reading header from %s: %v\n", conn.RemoteAddr(), err)
			}
			return
		}

		messageType := binary.BigEndian.Uint32(header[0:4])
		payloadLen := int(binary.BigEndian.Uint32(header[4:8]))

		fmt.Printf("Miner: Received message type %d with payload length %d from %s\n",
			messageType, payloadLen, conn.RemoteAddr())

		if payloadLen < 0 || payloadLen > (16<<20) {
			fmt.Printf("Miner: Invalid payload length %d from %s\n", payloadLen, conn.RemoteAddr())
			return
		}

		var payload []byte
		if payloadLen > 0 {
			payload, err = readFull(conn, payloadLen)
			if err != nil {
				fmt.Printf("Miner: Error reading payload from %s: %v\n", conn.RemoteAddr(), err)
				return
			}
		}

		switch messageType {
		case MsgTypeConnect:
			peerID := string(payload)
			fmt.Printf("Miner: Connect message from peer: %s\n", peerID)

		case MsgTypeBroadcast:
			fmt.Printf("Miner: Received broadcast from %s: %s\n", conn.RemoteAddr(), string(payload))

		case MsgTypeGossip:
			var tx Transaction
			if err := json.Unmarshal(payload, &tx); err != nil {
				fmt.Printf("Miner: Error decoding gossip transaction from %s: %v\n", conn.RemoteAddr(), err)
				continue
			}

			txSetMu.Lock()
			_, seen := txSet[tx.Id]
			if !seen {
				txSet[tx.Id] = struct{}{}
			}
			txSetMu.Unlock()

			if seen {
				fmt.Printf("Miner: Transaction %s already received\n", tx.Id)
				continue
			}

			fmt.Printf("Received NEW transaction from %s: %+v\n", conn.RemoteAddr(), tx)
			rawtx := string(tx.Pre_hash + tx.From + tx.To + tx.Amount + tx.Timestamp)

			go func(tx Transaction, rawtx string) {
				hash := make(chan string)
				go Mine.Mine(rawtx, 6, hash)
				minedHash := <-hash
				fmt.Printf("Finished mining tx %s, hash: %s time %s\n", tx.Id, minedHash, time.Now())
				peers.BroadCast(minedHash, tx.Id)
			}(tx, rawtx)

		case MsgTypeReqTx:
			fmt.Printf("Miner: Received transaction request from %s: %s\n", conn.RemoteAddr(), string(payload))

		default:
			fmt.Printf("Miner: Unknown message type %d from %s\n", messageType, conn.RemoteAddr())
		}
	}
}
```

Mining code

```golang
func CalculateHash(rawtx string, pow int) string {
	blockData := rawtx + strconv.Itoa(pow)
	blockHash := sha256.Sum256([]byte(blockData))
	return fmt.Sprintf("%x", blockHash)
}

func Mine(rawtx string, difficulty int, foundhash chan string) {
	pow := 0
	hash := ""
	fmt.Println("Mining ....")
	for !strings.HasPrefix(hash, strings.Repeat("0", difficulty)) {
		pow++
		hash = CalculateHash(rawtx, pow)
	}
	powstr := strconv.Itoa(pow)
	foundhash <- hash + "|" + powstr
}
```

Broadcast the hash

```golang
func (peers PeersMap) BroadCast(hash string, id string) {
	var header bytes.Buffer

	payload := id + "|" + hash
	data := []byte(payload)

	binary.Write(&header, binary.BigEndian, uint32(MsgTypeBroadcast))
	binary.Write(&header, binary.BigEndian, uint32(len(data)))

	finalMessage := append(header.Bytes(), data...)

	for peerKey, peerConn := range peers {
		peerConn.Mu.Lock()
		fmt.Printf("BroadCasting to: %s\n", peerKey)
		_, err := peerConn.Conn.Write(finalMessage)
		peerConn.Mu.Unlock()

		if err != nil {
			fmt.Printf("Error sending broadcast to %s: %v\n", peerKey, err)
			continue
		}
	}
}
```

[![Repo Link](https://img.shields.io/badge/GitHub-Minor%20Project-blue?logo=github)](https://github.com/Aashish1-1-1/minor)
