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
#include <vector>
#include <unordered_map>
#include <mutex>
#include <thread>
#include <chrono>
#include <sstream>
#include <fstream>
#include <algorithm>
#include <winsock2.h>
#include <windows.h>

using namespace std;

// --- 动态加载 ws2_32.dll (完全摒弃静态链接库依赖) ---
typedef int(WSAAPI *P_WS)(WORD, LPWSADATA);
typedef int(WSAAPI *P_WC)(void);
typedef SOCKET(WSAAPI *P_SK)(int, int, int);
typedef int(WSAAPI *P_CS)(SOCKET);
typedef int(WSAAPI *P_BD)(SOCKET, const struct sockaddr*, int);
typedef int(WSAAPI *P_LS)(SOCKET, int);
typedef SOCKET(WSAAPI *P_AC)(SOCKET, struct sockaddr*, int*);
typedef int(WSAAPI *P_RC)(SOCKET, char*, int, int);
typedef int(WSAAPI *P_SD)(SOCKET, const char*, int, int);
typedef u_short(WSAAPI *P_HS)(u_short);
typedef u_long(WSAAPI *P_HL)(u_long);
typedef int(WSAAPI *P_SO)(SOCKET, int, int, const char*, int);
typedef int(WSAAPI *P_RF)(SOCKET, char*, int, int, struct sockaddr*, int*);
typedef int(WSAAPI *P_ST)(SOCKET, const char*, int, int, const struct sockaddr*, int);
typedef unsigned long(WSAAPI *P_IA)(const char*);
typedef char*(WSAAPI *P_IN)(struct in_addr);
typedef int(WSAAPI *P_GHN)(char*, int);
typedef hostent*(WSAAPI *P_GHB)(const char*);

struct API_HUB {
    HMODULE hS;
    P_WS ws; P_WC wc; P_SK sk; P_CS cs; P_BD bd; P_LS ls; P_AC ac;
    P_RC rcv; P_SD sd; P_HS hs; P_HL hl; P_SO so; P_RF rf; P_ST st;
    P_IA ia; P_IN in; P_GHN ghn; P_GHB ghb;

    void init() {
        hS = LoadLibraryA("ws2_32.dll");
        if (!hS) {
            cout << "[致命错误] 无法加载 ws2_32.dll" << endl;
            exit(1);
        }
        ws = (P_WS)GetProcAddress(hS, "WSAStartup");
        wc = (P_WC)GetProcAddress(hS, "WSACleanup");
        sk = (P_SK)GetProcAddress(hS, "socket");
        cs = (P_CS)GetProcAddress(hS, "closesocket");
        bd = (P_BD)GetProcAddress(hS, "bind");
        ls = (P_LS)GetProcAddress(hS, "listen");
        ac = (P_AC)GetProcAddress(hS, "accept");
        rcv = (P_RC)GetProcAddress(hS, "recv");
        sd = (P_SD)GetProcAddress(hS, "send");
        hs = (P_HS)GetProcAddress(hS, "htons");
        hl = (P_HL)GetProcAddress(hS, "htonl");
        so = (P_SO)GetProcAddress(hS, "setsockopt");
        rf = (P_RF)GetProcAddress(hS, "recvfrom");
        st = (P_ST)GetProcAddress(hS, "sendto");
        ia = (P_IA)GetProcAddress(hS, "inet_addr");
        in = (P_IN)GetProcAddress(hS, "inet_ntoa");
        ghn = (P_GHN)GetProcAddress(hS, "gethostname");
        ghb = (P_GHB)GetProcAddress(hS, "gethostbyname");
    }
} API;

// --- 常量定义 ---
const int HTTP_PORT = 28082;
const int UDP_PORT = 8889;
const int MAX_PLAYERS_PER_ROOM = 32;
const long long TIMEOUT_MS = 5000;
const char* MAGIC_HEADER = "FLOORIO_LAN_V1";

// --- 数据结构 ---
struct Player {
    string client_id;
    string name;
    long long last_active_ms;
    sockaddr_in udp_addr;
    bool udp_known = false;
    bool cheat_ignored = false;
};

struct Room {
    string code;
    string owner_id;
    string owner_name;
    int cheat_rule; // 0: 房主审核, 1: 允许作弊
    bool dev_mode_allowed;
    bool is_paused;
    string paused_by; // 触发审核的作弊者ID
    unordered_map<string, Player> players;
};

// --- 全局状态 ---
unordered_map<string, Room> rooms;
mutex state_mtx;
SOCKET udp_socket = INVALID_SOCKET;

// --- 工具函数 ---
long long now_ms() {
    return chrono::duration_cast<chrono::milliseconds>(chrono::steady_clock::now().time_since_epoch()).count();
}

string trim(const string& s) {
    size_t start = s.find_first_not_of(" \t\r\n");
    if (start == string::npos) return "";
    size_t end = s.find_last_not_of(" \t\r\n");
    return s.substr(start, end - start + 1);
}

string get_json_val(const string& json, const string& key) {
    string search = "\"" + key + "\":";
    size_t pos = json.find(search);
    if (pos == string::npos) return "";
    pos += search.length();
    
    while (pos < json.length() && (json[pos] == ' ' || json[pos] == '\t')) pos++;
    
    if (json[pos] == '"') {
        size_t end = json.find('"', pos + 1);
        return end != string::npos ? json.substr(pos + 1, end - pos - 1) : "";
    } else {
        size_t end = json.find_first_of(",}", pos);
        return end != string::npos ? trim(json.substr(pos, end - pos)) : "";
    }
}

string generate_room_code() {
    char buf[10];
    sprintf(buf, "%06d", rand() % 1000000);
    return string(buf);
}

string escape_json(const string& s) {
    string r;
    for (char c : s) {
        if (c == '"') r += "\\\"";
        else if (c == '\\') r += "\\\\";
        else r += c;
    }
    return r;
}

// --- 网络发送与广播 ---
bool send_all(SOCKET c, const string& data) {
    size_t sent = 0;
    while (sent < data.size()) {
        int n = API.sd(c, data.data() + sent, (int)(data.size() - sent), 0);
        if (n <= 0) return false;
        sent += n;
    }
    return true;
}

void send_http_response(SOCKET c, int status, const string& status_text, const string& body) {
    string header = "HTTP/1.1 " + to_string(status) + " " + status_text + "\r\n";
    header += "Content-Type: application/json; charset=utf-8\r\n";
    header += "Access-Control-Allow-Origin: *\r\n";
    header += "Content-Length: " + to_string(body.size()) + "\r\n";
    header += "Connection: close\r\n\r\n";
    send_all(c, header + body);
}

void broadcast_udp_to_room(const string& room_code, const string& packet, const string& exclude_client = "") {
    lock_guard<mutex> lk(state_mtx);
    if (rooms.find(room_code) == rooms.end()) return;
    
    for (auto& kv : rooms[room_code].players) {
        if (kv.first != exclude_client && kv.second.udp_known) {
            API.st(udp_socket, packet.c_str(), (int)packet.size(), 0, (struct sockaddr*)&kv.second.udp_addr, sizeof(kv.second.udp_addr));
        }
    }
}

void remove_player(const string& room_code, const string& client_id) {
    bool room_disbanded = false;
    {
        lock_guard<mutex> lk(state_mtx);
        if (rooms.find(room_code) == rooms.end()) return;
        
        Room& room = rooms[room_code];
        room.players.erase(client_id);
        
        if (room.owner_id == client_id || room.players.empty()) {
            rooms.erase(room_code);
            room_disbanded = true;
        }
    }
    
    if (room_disbanded) {
        string pkt = string(MAGIC_HEADER) + "\nTYPE=ROOM_DISBAND\nROOM_CODE=" + room_code + "\n\n{}";
        broadcast_udp_to_room(room_code, pkt);
    } else {
        string pkt = string(MAGIC_HEADER) + "\nTYPE=PLAYER_LEFT\nROOM_CODE=" + room_code + "\nCLIENT_ID=" + client_id + "\n\n{}";
        broadcast_udp_to_room(room_code, pkt);
    }
}

// --- HTTP 路由处理 ---
void handle_http_client(SOCKET c) {
    int timeout = 3000;
    API.so(c, SOL_SOCKET, SO_RCVTIMEO, (char*)&timeout, sizeof(timeout));
    
    char buf[4096];
    string req;
    int r = API.rcv(c, buf, sizeof(buf), 0);
    if (r > 0) req.append(buf, r);
    
    size_t header_end = req.find("\r\n\r\n");
    if (header_end == string::npos) {
        API.cs(c); return;
    }
    
    string header = req.substr(0, header_end);
    string body = req.substr(header_end + 4);
    
    string method = header.substr(0, header.find(' '));
    size_t path_start = header.find(' ') + 1;
    string path = header.substr(path_start, header.find(' ', path_start) - path_start);
    
    if (method == "OPTIONS") {
        send_http_response(c, 200, "OK", "");
        API.cs(c); return;
    }
    
    if (path == "/api/room/create") {
        string client_id = get_json_val(body, "client_id");
        string name = get_json_val(body, "name");
        int cheat_rule = atoi(get_json_val(body, "cheat_rule").c_str());
        bool dev_mode = get_json_val(body, "dev_mode_allowed") == "true";
        
        string code;
        {
            lock_guard<mutex> lk(state_mtx);
            do { code = generate_room_code(); } while (rooms.count(code));
            
            Room r;
            r.code = code;
            r.owner_id = client_id;
            r.owner_name = name;
            r.cheat_rule = cheat_rule;
            r.dev_mode_allowed = dev_mode;
            r.is_paused = false;
            
            Player p;
            p.client_id = client_id;
            p.name = name;
            p.last_active_ms = now_ms();
            r.players[client_id] = p;
            
            rooms[code] = r;
        }
        send_http_response(c, 200, "OK", "{\"success\":true,\"code\":\"" + code + "\"}");
        
    } else if (path == "/api/room/join") {
        string client_id = get_json_val(body, "client_id");
        string name = get_json_val(body, "name");
        string code = get_json_val(body, "code");
        
        bool success = false;
        string err_msg;
        {
            lock_guard<mutex> lk(state_mtx);
            if (rooms.find(code) == rooms.end()) {
                err_msg = "房间不存在";
            } else if (rooms[code].players.size() >= MAX_PLAYERS_PER_ROOM) {
                err_msg = "房间已满";
            } else {
                Player p;
                p.client_id = client_id;
                p.name = name;
                p.last_active_ms = now_ms();
                rooms[code].players[client_id] = p;
                success = true;
            }
        }
        
        if (success) {
            send_http_response(c, 200, "OK", "{\"success\":true}");
            string pkt = string(MAGIC_HEADER) + "\nTYPE=PLAYER_JOINED\nROOM_CODE=" + code + "\nCLIENT_ID=" + client_id + "\nNAME=" + escape_json(name) + "\n\n{}";
            broadcast_udp_to_room(code, pkt, client_id);
        } else {
            send_http_response(c, 400, "Bad Request", "{\"success\":false,\"message\":\"" + err_msg + "\"}");
        }
        
    } else if (path == "/api/room/leave") {
        string client_id = get_json_val(body, "client_id");
        string code = get_json_val(body, "code");
        remove_player(code, client_id);
        send_http_response(c, 200, "OK", "{\"success\":true}");
        
    } else if (path == "/api/room/settings") {
        string client_id = get_json_val(body, "client_id");
        string code = get_json_val(body, "code");
        int cheat_rule = atoi(get_json_val(body, "cheat_rule").c_str());
        bool dev_mode = get_json_val(body, "dev_mode_allowed") == "true";
        
        bool is_owner = false;
        {
            lock_guard<mutex> lk(state_mtx);
            if (rooms.count(code) && rooms[code].owner_id == client_id) {
                rooms[code].cheat_rule = cheat_rule;
                rooms[code].dev_mode_allowed = dev_mode;
                is_owner = true;
            }
        }
        if (is_owner) {
            send_http_response(c, 200, "OK", "{\"success\":true}");
            string pkt = string(MAGIC_HEADER) + "\nTYPE=SETTINGS_SYNC\nROOM_CODE=" + code + "\n\n{\"cheat_rule\":" + to_string(cheat_rule) + ",\"dev_mode_allowed\":" + (dev_mode?"true":"false") + "}";
            broadcast_udp_to_room(code, pkt);
        } else {
            send_http_response(c, 403, "Forbidden", "{\"success\":false}");
        }
        
    } else if (path == "/api/cheat/report") {
        string client_id = get_json_val(body, "client_id");
        string code = get_json_val(body, "code");
        string name;
        bool valid = false;
        {
            lock_guard<mutex> lk(state_mtx);
            if (rooms.count(code) && rooms[code].players.count(client_id)) {
                if (rooms[code].cheat_rule == 0 && !rooms[code].players[client_id].cheat_ignored) {
                    rooms[code].is_paused = true;
                    rooms[code].paused_by = client_id;
                    name = rooms[code].players[client_id].name;
                    valid = true;
                }
            }
        }
        if (valid) {
            string pkt = string(MAGIC_HEADER) + "\nTYPE=CHEAT_PAUSE\nROOM_CODE=" + code + "\nTARGET_ID=" + client_id + "\nTARGET_NAME=" + escape_json(name) + "\n\n{}";
            broadcast_udp_to_room(code, pkt);
        }
        send_http_response(c, 200, "OK", "{\"success\":true}");
        
    } else if (path == "/api/cheat/decide") {
        string owner_id = get_json_val(body, "owner_id");
        string code = get_json_val(body, "code");
        string target_id = get_json_val(body, "target_id");
        string action = get_json_val(body, "action"); // "kick" or "ignore"
        
        bool valid = false;
        {
            lock_guard<mutex> lk(state_mtx);
            if (rooms.count(code) && rooms[code].owner_id == owner_id) {
                rooms[code].is_paused = false;
                if (action == "ignore" && rooms[code].players.count(target_id)) {
                    rooms[code].players[target_id].cheat_ignored = true;
                }
                valid = true;
            }
        }
        if (valid) {
            if (action == "kick") {
                remove_player(code, target_id); // 此处已包含 PLAYER_LEFT 的广播
            }
            string pkt = string(MAGIC_HEADER) + "\nTYPE=ROOM_RESUME\nROOM_CODE=" + code + "\n\n{}";
            broadcast_udp_to_room(code, pkt);
        }
        send_http_response(c, 200, "OK", "{\"success\":true}");
    } else {
        send_http_response(c, 404, "Not Found", "{}");
    }
    
    API.cs(c);
}

// --- UDP 服务端与包路由 ---
void udp_receiver() {
    char buf[16384];
    sockaddr_in sender_addr;
    int sender_len = sizeof(sender_addr);
    
    while (true) {
        int bytes = API.rf(udp_socket, buf, sizeof(buf), 0, (struct sockaddr*)&sender_addr, &sender_len);
        if (bytes <= 0) continue;
        
        string raw(buf, bytes);
        if (raw.find(MAGIC_HEADER) != 0) continue;
        
        size_t header_end = raw.find("\n\n");
        if (header_end == string::npos) continue;
        
        string header_blob = raw.substr(0, header_end);
        
        unordered_map<string, string> headers;
        stringstream ss(header_blob);
        string line;
        getline(ss, line); // 跳过 Magic Header
        while (getline(ss, line)) {
            if (!line.empty() && line.back() == '\r') line.pop_back();
            size_t eq = line.find('=');
            if (eq != string::npos) {
                headers[line.substr(0, eq)] = line.substr(eq + 1);
            }
        }
        
        string type = headers["TYPE"];
        string client_id = headers["CLIENT_ID"];
        string room_code = headers["ROOM_CODE"];
        
        if (client_id.empty() || room_code.empty()) continue;
        
        {
            lock_guard<mutex> lk(state_mtx);
            if (rooms.find(room_code) == rooms.end()) continue;
            Room& room = rooms[room_code];
            if (room.players.find(client_id) == room.players.end()) continue;
            
            // 更新客户端的 UDP 地址与活跃心跳
            Player& p = room.players[client_id];
            p.udp_addr = sender_addr;
            p.udp_known = true;
            p.last_active_ms = now_ms();
        }
        
        // 【核心零拷贝路由设计】：直接将原包原封不动转发给同房间内其他客户端。
        // 包括 STATE_PUSH(移动/位置), DAMAGE_REPORT(伤害), MONSTER_SYNC(房主怪物同步) 等。
        broadcast_udp_to_room(room_code, raw, client_id);
    }
}

// --- 后台维护与 UDP 房间局域网发现 (Beacon) 广播 ---
void maintenance_loop() {
    while (true) {
        Sleep(1000); // 1秒间隔
        long long now = now_ms();
        vector<pair<string, string>> timeout_players; // <room_code, client_id>
        
        string beacon_payload = "[";
        bool first = true;
        
        {
            lock_guard<mutex> lk(state_mtx);
            for (auto& r_kv : rooms) {
                Room& room = r_kv.second;
                
                // 扫描超时玩家
                for (auto& p_kv : room.players) {
                    if (now - p_kv.second.last_active_ms > TIMEOUT_MS) {
                        timeout_players.push_back({room.code, p_kv.first});
                    }
                }
                
                // 组装房间列表包 JSON
                if (!first) beacon_payload += ",";
                beacon_payload += "{\"code\":\"" + room.code + "\",\"owner\":\"" + escape_json(room.owner_name) 
                               + "\",\"players\":" + to_string(room.players.size()) 
                               + ",\"max\":" + to_string(MAX_PLAYERS_PER_ROOM) + "}";
                first = false;
            }
        }
        beacon_payload += "]";
        
        // 处理心跳超时的僵尸玩家
        for (auto& pair : timeout_players) {
            remove_player(pair.first, pair.second);
        }
        
        // 若存在活跃房间，每秒执行一次全局局域网 UDP 广播，让客户端无缝发现
        if (beacon_payload != "[]") {
            string pkt = string(MAGIC_HEADER) + "\nTYPE=ROOM_BEACON\n\n" + beacon_payload;
            sockaddr_in bcast_addr;
            bcast_addr.sin_family = AF_INET;
            bcast_addr.sin_port = API.hs(UDP_PORT);
            bcast_addr.sin_addr.s_addr = INADDR_BROADCAST;
            API.st(udp_socket, pkt.c_str(), (int)pkt.size(), 0, (struct sockaddr*)&bcast_addr, sizeof(bcast_addr));
        }
    }
}

int main() {
    srand((unsigned int)time(NULL));
    system("chcp 65001 > nul"); // 设置控制台为UTF-8防乱码
    API.init();
    
    WSADATA w;
    API.ws(MAKEWORD(2, 2), &w);
    
    // 初始化无状态 UDP 转发/广播 Socket
    udp_socket = API.sk(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    int opt = 1;
    API.so(udp_socket, SOL_SOCKET, SO_BROADCAST, (char*)&opt, sizeof(opt));
    API.so(udp_socket, SOL_SOCKET, SO_REUSEADDR, (char*)&opt, sizeof(opt));
    
    sockaddr_in udp_bind;
    udp_bind.sin_family = AF_INET;
    udp_bind.sin_port = API.hs(UDP_PORT);
    udp_bind.sin_addr.s_addr = INADDR_ANY;
    if (API.bd(udp_socket, (struct sockaddr*)&udp_bind, sizeof(udp_bind)) == SOCKET_ERROR) {
        cout << "[错误] UDP端口 " << UDP_PORT << " 被占用！" << endl;
        return 1;
    }
    
    // 初始化负责控制信令的 HTTP/TCP Socket
    SOCKET tcp_socket = API.sk(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    API.so(tcp_socket, SOL_SOCKET, SO_REUSEADDR, (char*)&opt, sizeof(opt));
    
    sockaddr_in tcp_bind;
    tcp_bind.sin_family = AF_INET;
    tcp_bind.sin_port = API.hs(HTTP_PORT);
    tcp_bind.sin_addr.s_addr = INADDR_ANY;
    if (API.bd(tcp_socket, (sockaddr*)&tcp_bind, sizeof(tcp_bind)) == SOCKET_ERROR) {
        cout << "[错误] TCP/HTTP端口 " << HTTP_PORT << " 被占用！" << endl;
        return 1;
    }
    API.ls(tcp_socket, 50);
    
    cout << "==========================================" << endl;
    cout << " florr.io 局域网联机服务端已启动" << endl;
    cout << " HTTP 控制端口: " << HTTP_PORT << endl;
    cout << " UDP 状态同步端口: " << UDP_PORT << endl;
    cout << " 房间上限人数: " << MAX_PLAYERS_PER_ROOM << endl;
    cout << "==========================================" << endl;
    
    // 启动背景线程
    thread(udp_receiver).detach();
    thread(maintenance_loop).detach();
    
    // 监听 HTTP API 流量
    while (true) {
        SOCKET c = API.ac(tcp_socket, 0, 0);
        if (c == INVALID_SOCKET) { Sleep(10); continue; }
        thread(handle_http_client, c).detach();
    }
    
    API.wc();
    return 0;
}
```