---
title: florr.oi联机服务端
published: 2026-01-09
description: "v1.0"
image: "./cover.jpeg"
tags: ["C++"]
category: 程序
draft: false
---

```cpp
//florr.oi联机服务端 - v1.0
#include <iostream>
#include <string>
#include <thread>
#include <vector>
#include <map>
#include <mutex>
#include <winsock2.h>
#include <ws2tcpip.h>
#include <windows.h>

using namespace std;

// --- 动态加载 ws2_32.dll ---
typedef int(WSAAPI *P_WS)(WORD, LPWSADATA);
typedef int(WSAAPI *P_WC)(void);
typedef SOCKET(WSAAPI *P_SK)(int, int, int);
typedef int(WSAAPI *P_CS)(SOCKET);
typedef int(WSAAPI *P_BD)(SOCKET, const struct sockaddr*, int);
typedef int(WSAAPI *P_LS)(SOCKET, int);
typedef SOCKET(WSAAPI *P_AC)(SOCKET, struct sockaddr*, int*);
typedef int(WSAAPI *P_RC)(SOCKET, char*, int, int);
typedef int(WSAAPI *P_SD)(SOCKET, const char*, int, int);
typedef int(WSAAPI *P_ST)(SOCKET, const char*, int, int, const struct sockaddr*, int);
typedef u_short(WSAAPI *P_HS)(u_short);
typedef int(WSAAPI *P_SO)(SOCKET, int, int, const char*, int);
typedef char*(WSAAPI *P_IN)(struct in_addr);
typedef int(WSAAPI *P_GHN)(char*, int);
typedef hostent*(WSAAPI *P_GHB)(const char*);

struct API_HUB {
    HMODULE hS;
    P_WS ws; P_WC wc; P_SK sk; P_CS cs; P_BD bd;
    P_LS ls; P_AC ac; P_RC rcv; P_SD sd; P_ST st;
    P_HS hs; P_SO so; P_IN in; P_GHN ghn; P_GHB ghb;

    void init() {
        hS = LoadLibraryA("ws2_32.dll");
        if (hS) {
            ws = (P_WS)GetProcAddress(hS, "WSAStartup");
            wc = (P_WC)GetProcAddress(hS, "WSACleanup");
            sk = (P_SK)GetProcAddress(hS, "socket");
            cs = (P_CS)GetProcAddress(hS, "closesocket");
            bd = (P_BD)GetProcAddress(hS, "bind");
            ls = (P_LS)GetProcAddress(hS, "listen");
            ac = (P_AC)GetProcAddress(hS, "accept");
            rcv = (P_RC)GetProcAddress(hS, "recv");
            sd = (P_SD)GetProcAddress(hS, "send");
            st = (P_ST)GetProcAddress(hS, "sendto");
            hs = (P_HS)GetProcAddress(hS, "htons");
            so = (P_SO)GetProcAddress(hS, "setsockopt");
            in = (P_IN)GetProcAddress(hS, "inet_ntoa");
            ghn = (P_GHN)GetProcAddress(hS, "gethostname");
            ghb = (P_GHB)GetProcAddress(hS, "gethostbyname");
        }
    }
} API;

// --- 数据结构 ---
struct Player {
    string id, name;
    double x = 0, y = 0, hp = 100, maxHp = 100;
    int lvl = 1;
    bool isAlive = true, isBleeding = false;
    long long lastHeartbeat = 0;
};

struct Room {
    string code, hostId, hostName;
    int cheatRule = 0;
    bool devAllowed = true;
    bool isPaused = false;
    map<string, Player> players;
};

map<string, Room> rooms;
mutex roomMutex;

// --- 工具函数：提取简易JSON字段 ---
string getJsonString(const string& json, const string& key) {
    size_t pos = json.find("\"" + key + "\":");
    if (pos == string::npos) return "";
    pos += key.length() + 3;
    size_t start = json.find("\"", pos);
    if (start == string::npos) return "";
    size_t end = json.find("\"", start + 1);
    return json.substr(start + 1, end - start - 1);
}

double getJsonDouble(const string& json, const string& key) {
    size_t pos = json.find("\"" + key + "\":");
    if (pos == string::npos) return 0.0;
    pos += key.length() + 3;
    size_t end = json.find_first_of(",}", pos);
    if (end == string::npos) return 0.0;
    string valStr = json.substr(pos, end - pos);
    try {
        return stod(valStr);
    } catch (...) {
        return 0.0; 
    }
}

bool getJsonBool(const string& json, const string& key) {
    size_t pos = json.find("\"" + key + "\":");
    if (pos == string::npos) return false;
    pos += key.length() + 3;
    return (json.substr(pos, 4) == "true");
}

long long getTimestamp() {
    return GetTickCount64();
}

// --- HTTP 响应构造 ---
void sendHttpResponse(SOCKET clientSocket, const string& jsonBody) {
    string response = "HTTP/1.1 200 OK\r\n"
                      "Content-Type: application/json; charset=utf-8\r\n"
                      "Access-Control-Allow-Origin: *\r\n"
                      "Access-Control-Allow-Headers: Content-Type\r\n"
                      "Connection: close\r\n"
                      "Content-Length: " + to_string(jsonBody.length()) + "\r\n\r\n" + jsonBody;
    API.sd(clientSocket, response.c_str(), (int)response.length(), 0);
}

// --- 处理客户端 HTTP 请求 ---
void handleClient(SOCKET clientSocket) {
    char buffer[4096] = {0};
    API.rcv(clientSocket, buffer, (int)sizeof(buffer) - 1, 0);
    string request(buffer);

    if (request.find("OPTIONS") == 0) {
        sendHttpResponse(clientSocket, "{}");
        API.cs(clientSocket);
        return;
    }

    size_t bodyPos = request.find("\r\n\r\n");
    string body = (bodyPos != string::npos) ? request.substr(bodyPos + 4) : "";

    if (request.find("POST /api/room/create") != string::npos) {
        string clientId = getJsonString(body, "client_id");
        string name = getJsonString(body, "name");
        
        lock_guard<mutex> lock(roomMutex);
        string code = to_string(100000 + rand() % 900000); 
        Room r; r.code = code; r.hostId = clientId; r.hostName = name;
        Player p; p.id = clientId; p.name = name; p.lastHeartbeat = getTimestamp();
        r.players[clientId] = p;
        rooms[code] = r;
        
        sendHttpResponse(clientSocket, "{\"success\":true, \"code\":\"" + code + "\"}");
    } 
    else if (request.find("POST /api/room/join") != string::npos) {
        string clientId = getJsonString(body, "client_id");
        string code = getJsonString(body, "code");
        string name = getJsonString(body, "name");

        lock_guard<mutex> lock(roomMutex);
        if (rooms.count(code) > 0 && rooms[code].players.size() < 32) {
            Player p; p.id = clientId; p.name = name; p.lastHeartbeat = getTimestamp();
            rooms[code].players[clientId] = p;
            sendHttpResponse(clientSocket, "{\"success\":true}");
        } else {
            sendHttpResponse(clientSocket, "{\"success\":false, \"message\":\"房间不存在或已满\"}");
        }
    }
    else if (request.find("GET /api/room/list") != string::npos) { 
        lock_guard<mutex> lock(roomMutex);
        string res = "{\"rooms\":[";
        bool first = true;
        for (auto& pair : rooms) {
            if (!first) res += ",";
            res += "{\"code\":\"" + pair.first + "\",\"name\":\"" + pair.second.hostName + "\",\"count\":" + to_string(pair.second.players.size()) + "}";
            first = false;
        }
        res += "]}";
        sendHttpResponse(clientSocket, res);
    }
    else if (request.find("POST /api/state") != string::npos) {
        string clientId = getJsonString(body, "client_id");
        string code = getJsonString(body, "room_code");

        lock_guard<mutex> lock(roomMutex);
        if (rooms.count(code)) {
            auto& p = rooms[code].players[clientId];
            p.x = getJsonDouble(body, "x"); p.y = getJsonDouble(body, "y");p.hp = getJsonDouble(body, "hp");
            p.isBleeding = getJsonBool(body, "isBleeding");
            p.lastHeartbeat = getTimestamp();

            string res = "{\"success\":true, \"isPaused\":" + string(rooms[code].isPaused ? "true" : "false") + ", \"players\":{";
            bool first = true;
            for (auto& other : rooms[code].players) {
                if (other.first == clientId) continue; 
                if (getTimestamp() - other.second.lastHeartbeat > 5000) continue; 
                
                if (!first) res += ",";
                res += "\"" + other.first + "\":{\"x\":" + to_string(other.second.x) + ",\"y\":" + to_string(other.second.y) + 
                       ",\"hp\":" + to_string(other.second.hp) + ",\"isBleeding\":" + (other.second.isBleeding ? "true" : "false") + 
                       ",\"name\":\"" + other.second.name + "\"}";
                first = false;
            }
            res += "}}";
            sendHttpResponse(clientSocket, res);
        } else {
            sendHttpResponse(clientSocket, "{\"success\":false}");
        }
    }
    API.cs(clientSocket);
}

// --- TCP 监听线程 ---
void tcpServer() {
    SOCKET listenSocket = API.sk(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    sockaddr_in serverAddr = {0};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = API.hs(28082);
    serverAddr.sin_addr.s_addr = INADDR_ANY;

    API.bd(listenSocket, (SOCKADDR*)&serverAddr, sizeof(serverAddr));
    API.ls(listenSocket, SOMAXCONN);
    
    cout << "[INFO] IP : ";
    char hostName[256];
    API.ghn(hostName, sizeof(hostName));
    struct hostent* host = API.ghb(hostName);
    for (int i = 0; host->h_addr_list[i] != NULL; i++) {
        cout << API.in(*(struct in_addr*)host->h_addr_list[i]) << " ";
    }
    cout << endl;

    while (true) {
        SOCKET clientSocket = API.ac(listenSocket, NULL, NULL);
        if (clientSocket != INVALID_SOCKET) {
            thread(handleClient, clientSocket).detach();
        }
    }
}

// --- UDP 广播发现线程 ---
void udpBeacon() {
    SOCKET udpSocket = API.sk(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    BOOL broadcast = TRUE;
    API.so(udpSocket, SOL_SOCKET, SO_BROADCAST, (char*)&broadcast, sizeof(broadcast));

    sockaddr_in bcastAddr = {0};
    bcastAddr.sin_family = AF_INET;
    bcastAddr.sin_port = API.hs(8889);
    bcastAddr.sin_addr.s_addr = INADDR_BROADCAST;

    while (true) {
        Sleep(1500); 
        lock_guard<mutex> lock(roomMutex);
        for (auto& pair : rooms) {
            string msg = "FLOORIO_LAN_V1|TYPE:ROOM_BEACON|CODE:" + pair.first + "|PLAYERS:" + to_string(pair.second.players.size());
            API.st(udpSocket, msg.c_str(), (int)msg.length(), 0, (SOCKADDR*)&bcastAddr, sizeof(bcastAddr));
        }
    }
}

int main() {
    // 1. 先初始化函数指针动态获取
    API.init();

    // 2. 正常使用底层 API
    WSADATA wsaData;
    API.ws(MAKEWORD(2, 2), &wsaData);
    srand((unsigned int)GetTickCount());

    thread tcpThread(tcpServer);
    thread udpThread(udpBeacon);

    tcpThread.join();
    udpThread.join();
    
    API.wc();
    return 0;
}
```