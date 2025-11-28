# project_management_tool23435

import * as http from 'http';
import * as https from 'https';
import * as net from 'net';
import * as dgram from 'dgram';
import * as WebSocket from 'ws';
import * as jwt from 'jsonwebtoken';
import * as cors from 'cors';
import * as rateLimit from 'express-rate-limit';
import { EventEmitter } from 'events';
import { URL } from 'url';

// For MQTT (optional - install mqtt package if needed)
// import * as mqtt from 'mqtt';

interface ValidationRule {
  field: string;
  type:
    | 'required'
    | 'email'
    | 'number'
    | 'minLength'
    | 'maxLength'
    | 'regex'
    | 'custom';
  value?: string | number;
  message: string;
}

interface PayloadSource {
  type: 'manual' | 'file' | 'generator';
  content?: string;
  file?: any;
  generator?: {
    template: string;
    count: number;
    fields: Record<string, any>;
  };
}

interface AuthConfig {
  type: 'none' | 'jwt' | 'bearer' | 'basic' | 'apikey' | 'custom';
  token?: string;
  secret?: string;
  algorithm?: string;
  expiresIn?: string;
  claims?: Record<string, any>;
  header?: string;
  prefix?: string;
}

interface EndpointConfig {
  id: string;
  method: string;
  path: string;
  protocol: string;
  response: string;
  status: number;
  headers: Record<string, string>;
  latency: number;
  auth: AuthConfig;
  validation: ValidationRule[];
  payloadSource: PayloadSource;
  conditionalResponses: Array<{
    condition: string;
    response: string;
    status: number;
  }>;
  rateLimit?: {
    enabled: boolean;
    requests: number;
    window: number;
  };
  cors?: {
    enabled: boolean;
    origins: string[];
    methods: string[];
    headers: string[];
  };
  error?: {
    status?: number;
    message?: string;
    probability?: number;
  };
  protocolConfig?: {
    mqtt?: {
      topic: string;
      qos: number;
      retain: boolean;
      clientId?: string;
      username?: string;
      password?: string;
      keepAlive?: number;
      cleanSession?: boolean;
    };
    websocket?: {
      subprotocol?: string;
      compression?: boolean;
      maxPayloadLength?: number;
      heartbeat?: {
        enabled: boolean;
        interval: number;
      };
    };
    tcp?: {
      host: string;
      port: number;
      type: string;
      encoding?: string;
      keepAlive?: boolean;
      timeout?: number;
    };
    udp?: {
      host: string;
      port: number;
      type: string;
      multicast?: {
        enabled: boolean;
        address: string;
      };
    };
    grpc?: {
      service: string;
      method: string;
      streamType: string;
      protoFile?: string;
      reflection?: boolean;
    };
    [key: string]: any;
  };
}

interface ServerConfig {
  port: number;
  endpoints: EndpointConfig[];
}

// Enterprise-grade simulated server with advanced multi-protocol support
class SimulatedServerService extends EventEmitter {
  private config: ServerConfig;
  private servers: any[];
  private httpServer: http.Server | null;
  private httpsServer: https.Server | null;
  private wsServer: WebSocket.Server | null;
  private wssServer: WebSocket.Server | null;
  private tcpServers: Map<string, net.Server>;
  private udpSockets: Map<string, dgram.Socket>;
  private logWSS: WebSocket.Server | null;
  private logClients: WebSocket[];
  private rateLimiters: Map<string, any>;
  private requestCounts: Map<string, { count: number; resetTime: number }>;
  private mqttClients: Map<string, any>;
  private sseClients: Map<string, http.ServerResponse[]>;

  // 🔑 NEW: Connection tracking for proper cleanup
  private httpConnections: Set<net.Socket> = new Set();
  private httpsConnections: Set<net.Socket> = new Set();
  private tcpConnections: Map<string, Set<net.Socket>> = new Map();
  private wsConnections: Set<WebSocket> = new Set();
  private isShuttingDown: boolean = false;

  constructor(config: ServerConfig) {
    super();
    this.config = config;
    this.servers = [];
    this.httpServer = null;
    this.httpsServer = null;
    this.wsServer = null;
    this.wssServer = null;
    this.tcpServers = new Map();
    this.udpSockets = new Map();
    this.logWSS = null;
    this.logClients = [];
    this.rateLimiters = new Map();
    this.requestCounts = new Map();
    this.mqttClients = new Map();
    this.sseClients = new Map();
  }

  /**
   * Track HTTP/HTTPS connections for proper cleanup
   */
  private trackConnections(
    server: http.Server | https.Server,
    connectionSet: Set<net.Socket>,
  ) {
    server.on('connection', (socket: net.Socket) => {
      connectionSet.add(socket);

      // Remove from set when closed naturally
      socket.on('close', () => {
        connectionSet.delete(socket);
      });

      // If already shutting down, close immediately
      if (this.isShuttingDown) {
        socket.destroy();
      }
    });
  }

  // async start() {
  //   if (this.httpServer) return;

  //   const { port, endpoints = [] } = this.config;
  //   console.log('port and end point', port, endpoints);
  //   const protocolsUsed = [...new Set(endpoints.map((ep) => ep.protocol))];

  //   this.log(`Starting multi-protocol server on port ${port}...`);
  //   this.log(`Protocols to initialize: ${protocolsUsed.join(', ')}`);

  //   // HTTP/HTTPS Server
  //   if (
  //     protocolsUsed.some((p) =>
  //       ['HTTP', 'HTTPS', 'GraphQL', 'Server-Sent Events (SSE)'].includes(p),
  //     )
  //   ) {
  //     this.httpServer = http.createServer(async (req, res) => {
  //       try {
  //         await this.handleHttpRequest(req, res, endpoints);
  //       } catch (error) {
  //         this.log(`Error handling HTTP request: ${error}`);
  //         res.statusCode = 500;
  //         res.end('Internal Server Error');
  //       }
  //     });

  //     this.httpServer.listen(port, () => {
  //       this.log(`HTTP server started on port ${port}`);
  //     });
  //     this.servers.push(this.httpServer);
  //   }

  //   // WebSocket Server
  //   if (protocolsUsed.some((p) => p.includes('WebSocket'))) {
  //     const wsPort = port + 1;
  //     this.wsServer = new WebSocket.Server({ port: wsPort });
  //     this.wsServer.on('connection', (ws: any, req: any) => {
  //       this.handleWebSocketConnection(ws, req, endpoints);
  //     });
  //     this.servers.push(this.wsServer);
  //     this.log(`WebSocket server started on port ${wsPort}`);
  //   }

  //   // TCP Servers
  //   const tcpEndpoints = endpoints.filter((ep) => ep.protocol === 'TCP');
  //   for (const endpoint of tcpEndpoints) {
  //     const tcpPort = endpoint.protocolConfig?.tcp?.port || port + 10;
  //     const tcpHost = endpoint.protocolConfig?.tcp?.host || '0.0.0.0';

  //     const tcpServer = net.createServer((socket) => {
  //       this.handleTcpConnection(socket, endpoint);
  //     });

  //     tcpServer.listen(tcpPort, tcpHost, () => {
  //       this.log(`TCP server started on ${tcpHost}:${tcpPort}`);
  //     });

  //     this.tcpServers.set(endpoint.id, tcpServer);
  //     this.servers.push(tcpServer);
  //   }

  //   // UDP Sockets
  //   const udpEndpoints = endpoints.filter((ep) => ep.protocol === 'UDP');
  //   for (const endpoint of udpEndpoints) {
  //     const udpPort = endpoint.protocolConfig?.udp?.port || port + 20;
  //     const udpHost = endpoint.protocolConfig?.udp?.host || '0.0.0.0';

  //     const udpSocket = dgram.createSocket('udp4');

  //     udpSocket.on('message', (msg, rinfo) => {
  //       this.handleUdpMessage(msg, rinfo, endpoint, udpSocket);
  //     });

  //     udpSocket.bind(udpPort, udpHost, () => {
  //       this.log(`UDP server started on ${udpHost}:${udpPort}`);
  //     });

  //     this.udpSockets.set(endpoint.id, udpSocket);
  //     this.servers.push(udpSocket);
  //   }

  //   // MQTT Simulation (simplified - in production, use actual MQTT broker)
  //   const mqttEndpoints = endpoints.filter(
  //     (ep) => ep.protocol === 'MQTT' || ep.protocol === 'MQTT over WebSocket',
  //   );
  //   if (mqttEndpoints.length > 0) {
  //     this.log('MQTT endpoints detected - starting MQTT simulation service');
  //     // Initialize MQTT message handling
  //     this.initializeMqttSimulation(mqttEndpoints);
  //   }

  //   // Initialize other protocols as needed
  //   await this.initializeOtherProtocols(endpoints, port);

  //   // WebSocket for log streaming (always start this)
  //   this.logWSS = new WebSocket.Server({ port: port + 2 });
  //   this.logWSS.on('connection', (ws: WebSocket) => {
  //     this.logClients.push(ws);
  //     ws.on('close', () => {
  //       this.logClients = this.logClients.filter((client) => client !== ws);
  //     });
  //     this.log('Log client connected');
  //   });
  //   this.servers.push(this.logWSS);

  //   this.log(
  //     `Multi-protocol server started successfully with ${protocolsUsed.length} protocol types`,
  //   );
  // }
  async start() {
    if (this.httpServer) {
      this.log('Server already running');
      return;
    }

    this.isShuttingDown = false;
    const { port, endpoints = [] } = this.config;
    const protocolsUsed = [...new Set(endpoints.map((ep) => ep.protocol))];

    this.log(`Starting multi-protocol server on port ${port}...`);
    this.log(`Protocols to initialize: ${protocolsUsed.join(', ')}`);

    // HTTP/HTTPS Server
    if (
      protocolsUsed.some((p) =>
        ['HTTP', 'HTTPS', 'GraphQL', 'Server-Sent Events (SSE)'].includes(p),
      )
    ) {
      this.httpServer = http.createServer(async (req, res) => {
        // Reject new requests during shutdown
        if (this.isShuttingDown) {
          res.statusCode = 503;
          res.end('Server is shutting down');
          return;
        }

        try {
          await this.handleHttpRequest(req, res, endpoints);
        } catch (error) {
          this.log(`Error handling HTTP request: ${error}`);
          res.statusCode = 500;
          res.end('Internal Server Error');
        }
      });

      // 🔑 Track connections
      this.trackConnections(this.httpServer, this.httpConnections);

      await new Promise<void>((resolve, reject) => {
        this.httpServer!.listen(port, () => {
          this.log(`HTTP server started on port ${port}`);
          resolve();
        });

        this.httpServer!.on('error', (error) => {
          this.log(`HTTP server error: ${error}`);
          reject(error);
        });
      });

      this.servers.push(this.httpServer);
    }

    // WebSocket Server
    if (protocolsUsed.some((p) => p.includes('WebSocket'))) {
      const wsPort = port + 1;
      this.wsServer = new WebSocket.Server({ port: wsPort });

      this.wsServer.on(
        'connection',
        (ws: WebSocket, req: http.IncomingMessage) => {
          // 🔑 Track WebSocket connection
          this.wsConnections.add(ws);

          ws.on('close', () => {
            this.wsConnections.delete(ws);
          });

          // Close immediately if shutting down
          if (this.isShuttingDown) {
            ws.close();
            return;
          }

          this.handleWebSocketConnection(ws, req, endpoints);
        },
      );

      this.servers.push(this.wsServer);
      this.log(`WebSocket server started on port ${wsPort}`);
    }

    // TCP Servers
    const tcpEndpoints = endpoints.filter((ep) => ep.protocol === 'TCP');
    for (const endpoint of tcpEndpoints) {
      const tcpPort = endpoint.protocolConfig?.tcp?.port || port + 10;
      const tcpHost = endpoint.protocolConfig?.tcp?.host || '0.0.0.0';

      const tcpServer = net.createServer((socket) => {
        // 🔑 Track TCP connection
        if (!this.tcpConnections.has(endpoint.id)) {
          this.tcpConnections.set(endpoint.id, new Set());
        }
        this.tcpConnections.get(endpoint.id)!.add(socket);

        socket.on('close', () => {
          this.tcpConnections.get(endpoint.id)?.delete(socket);
        });

        // Close immediately if shutting down
        if (this.isShuttingDown) {
          socket.destroy();
          return;
        }

        this.handleTcpConnection(socket, endpoint);
      });

      await new Promise<void>((resolve) => {
        tcpServer.listen(tcpPort, tcpHost, () => {
          this.log(`TCP server started on ${tcpHost}:${tcpPort}`);
          resolve();
        });
      });

      this.tcpServers.set(endpoint.id, tcpServer);
      this.servers.push(tcpServer);
    }

    // UDP Sockets
    const udpEndpoints = endpoints.filter((ep) => ep.protocol === 'UDP');
    for (const endpoint of udpEndpoints) {
      const udpPort = endpoint.protocolConfig?.udp?.port || port + 20;
      const udpHost = endpoint.protocolConfig?.udp?.host || '0.0.0.0';

      const udpSocket = dgram.createSocket('udp4');

      udpSocket.on('message', (msg, rinfo) => {
        if (!this.isShuttingDown) {
          this.handleUdpMessage(msg, rinfo, endpoint, udpSocket);
        }
      });

      await new Promise<void>((resolve) => {
        udpSocket.bind(udpPort, udpHost, () => {
          this.log(`UDP server started on ${udpHost}:${udpPort}`);
          resolve();
        });
      });

      this.udpSockets.set(endpoint.id, udpSocket);
      this.servers.push(udpSocket);
    }

    // MQTT Simulation
    const mqttEndpoints = endpoints.filter(
      (ep) => ep.protocol === 'MQTT' || ep.protocol === 'MQTT over WebSocket',
    );
    if (mqttEndpoints.length > 0) {
      this.log('MQTT endpoints detected - starting MQTT simulation service');
      this.initializeMqttSimulation(mqttEndpoints);
    }

    // Initialize other protocols
    await this.initializeOtherProtocols(endpoints, port);

    // WebSocket for log streaming
    this.logWSS = new WebSocket.Server({ port: port + 2 });
    this.logWSS.on('connection', (ws: WebSocket) => {
      this.logClients.push(ws);
      ws.on('close', () => {
        this.logClients = this.logClients.filter((client) => client !== ws);
      });
      this.log('Log client connected');
    });
    this.servers.push(this.logWSS);

    this.log(
      `✅ Multi-protocol server started successfully with ${protocolsUsed.length} protocol types`,
    );
  }

  private async handleHttpRequest(
    req: http.IncomingMessage,
    res: http.ServerResponse,
    endpoints: EndpointConfig[],
  ) {
    const method = req.method || 'GET';
    const url = req.url || '/';
    const parsedUrl = new URL(url, `http://${req.headers.host}`);
    const path = parsedUrl.pathname;

    // Find matching endpoint
    const endpoint = endpoints.find(
      (ep) =>
        ep.protocol === 'HTTP' &&
        ep.method === method &&
        this.matchPath(ep.path, path),
    );

    if (!endpoint) {
      res.statusCode = 404;
      res.end(JSON.stringify({ error: 'Endpoint not found' }));
      this.log(`404 - ${method} ${path}`);
      return;
    }

    this.log(`Handling ${method} ${path} - Endpoint: ${endpoint.id}`);

    // Rate limiting
    if (endpoint.rateLimit?.enabled && !this.checkRateLimit(endpoint, req)) {
      res.statusCode = 429;
      res.end(JSON.stringify({ error: 'Rate limit exceeded' }));
      this.log(`Rate limit exceeded for ${method} ${path}`);
      return;
    }

    // CORS handling
    if (endpoint.cors?.enabled) {
      this.applyCorsHeaders(res, endpoint.cors);
      if (method === 'OPTIONS') {
        res.statusCode = 200;
        res.end();
        return;
      }
    }

    // Parse request body
    const body = await this.parseRequestBody(req);
    const requestData = {
      method,
      path,
      headers: req.headers,
      query: Object.fromEntries(parsedUrl.searchParams),
      params: this.extractPathParams(endpoint.path, path),
      body,
    };

    // Authentication
    if (!this.validateAuth(endpoint.auth, req)) {
      res.statusCode = 401;
      res.end(JSON.stringify({ error: 'Authentication failed' }));
      this.log(`Authentication failed for ${method} ${path}`);
      return;
    }

    // Request validation
    const validationError = this.validateRequest(
      endpoint.validation,
      requestData,
    );
    if (validationError) {
      res.statusCode = 400;
      res.end(JSON.stringify({ error: validationError }));
      this.log(`Validation failed for ${method} ${path}: ${validationError}`);
      return;
    }

    // Error simulation
    if (
      endpoint.error?.probability &&
      Math.random() * 100 < endpoint.error.probability
    ) {
      res.statusCode = endpoint.error.status || 500;
      res.end(
        JSON.stringify({ error: endpoint.error.message || 'Simulated error' }),
      );
      this.log(`Simulated error for ${method} ${path}`);
      return;
    }

    // Latency simulation
    if (endpoint.latency > 0) {
      await new Promise((resolve) => setTimeout(resolve, endpoint.latency));
    }

    // Check conditional responses
    const conditionalResponse = this.evaluateConditionalResponses(
      endpoint.conditionalResponses,
      requestData,
    );

    let responseBody = conditionalResponse?.response || endpoint.response;
    let statusCode = conditionalResponse?.status || endpoint.status;

    // Process dynamic variables in response
    responseBody = this.processDynamicVariables(responseBody);

    // Set headers
    Object.entries(endpoint.headers).forEach(([key, value]) => {
      res.setHeader(key, value);
    });

    res.statusCode = statusCode;
    res.end(responseBody);

    this.log(`${statusCode} - ${method} ${path} - Response sent`);
  }

  // private handleWebSocketConnection(
  //   ws: WebSocket,
  //   req: http.IncomingMessage,
  //   endpoints: EndpointConfig[],
  // ) {
  //   const url = req.url || '/';

  //   ws.on('message', (message) => {
  //     try {
  //       const endpoint = endpoints.find((ep) => ep.protocol === 'WebSocket');
  //       if (!endpoint) {
  //         ws.send(
  //           JSON.stringify({ error: 'No WebSocket endpoint configured' }),
  //         );
  //         return;
  //       }

  //       // Simulate latency
  //       const sendResponse = () => {
  //         const response = this.processDynamicVariables(endpoint.response);
  //         ws.send(response);
  //         this.log(`WebSocket message sent to ${url}`);
  //       };

  //       if (endpoint.latency > 0) {
  //         setTimeout(sendResponse, endpoint.latency);
  //       } else {
  //         sendResponse();
  //       }
  //     } catch (error) {
  //       ws.send(JSON.stringify({ error: 'WebSocket error' }));
  //       this.log(`WebSocket error: ${error}`);
  //     }
  //   });

  //   this.log(`WebSocket connection established for ${url}`);
  // }

  private handleWebSocketConnection(
    ws: WebSocket,
    req: http.IncomingMessage,
    endpoints: EndpointConfig[],
  ) {
    // const url = req. + req.url || '/';
    const url = req.url

    // 1️⃣ Match endpoint by WebSocket + path
    const endpoint = endpoints.find(
      (ep) => ep.protocol === 'WebSocket' && ep.path === url,
    );

    if (!endpoint) {
      ws.send(
        JSON.stringify({
          error: `No WebSocket endpoint configured for ${url}`,
        }),
      );
      ws.close();
      this.log(
        `❌ WebSocket connection rejected for ${url} — no matching endpoint`,
      );
      return;
    }

    this.log(`🔗 WebSocket connection established for ${url}`);

    // 2️⃣ Handle WebSocket messages
    ws.on('message', (msg) => {
      this.log(`📥 WS message received on ${url}: ${msg}`);

      // Only respond if method is MESSAGE
      if (endpoint.method !== 'MESSAGE') {
        ws.send(
          JSON.stringify({
            error: `Unsupported WS method ${endpoint.method}`,
          }),
        );
        return;
      }

      // Build response
      let response = this.processDynamicVariables(endpoint.response);

      // Optional: Insert raw message as {{message}}
      response = response.replace('{{message}}', msg.toString());

      const sendResponse = () => {
        ws.send(response);
        this.log(`📤 WS response sent on ${url}`);
      };

      // Simulate latency
      if (endpoint.latency > 0) {
        setTimeout(sendResponse, endpoint.latency);
      } else {
        sendResponse();
      }
    });

    // 3️⃣ Handle disconnect
    ws.on('close', () => {
      this.log(`🔌 WebSocket disconnected for ${url}`);
    });
  }

  private matchPath(templatePath: string, actualPath: string): boolean {
    // Simple path matching with parameter support
    const templateParts = templatePath.split('/');
    const actualParts = actualPath.split('/');

    if (templateParts.length !== actualParts.length) return false;

    return templateParts.every((part, index) => {
      return part.startsWith(':') || part === actualParts[index];
    });
  }

  private extractPathParams(
    templatePath: string,
    actualPath: string,
  ): Record<string, string> {
    const templateParts = templatePath.split('/');
    const actualParts = actualPath.split('/');
    const params: Record<string, string> = {};

    templateParts.forEach((part, index) => {
      if (part.startsWith(':') && actualParts[index]) {
        params[part.slice(1)] = actualParts[index];
      }
    });

    return params;
  }

  private checkRateLimit(
    endpoint: EndpointConfig,
    req: http.IncomingMessage,
  ): boolean {
    if (!endpoint.rateLimit?.enabled) return true;

    const key = `${endpoint.id}_${req.connection.remoteAddress}`;
    const now = Date.now();
    const windowMs = endpoint.rateLimit.window * 1000;

    const current = this.requestCounts.get(key);

    if (!current || now > current.resetTime) {
      this.requestCounts.set(key, { count: 1, resetTime: now + windowMs });
      return true;
    }

    if (current.count >= endpoint.rateLimit.requests) {
      return false;
    }

    current.count++;
    return true;
  }

  private applyCorsHeaders(res: http.ServerResponse, cors: any) {
    res.setHeader('Access-Control-Allow-Origin', cors.origins.join(', '));
    res.setHeader('Access-Control-Allow-Methods', cors.methods.join(', '));
    res.setHeader('Access-Control-Allow-Headers', cors.headers.join(', '));
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }

  private async parseRequestBody(req: http.IncomingMessage): Promise<any> {
    return new Promise((resolve) => {
      let body = '';
      req.on('data', (chunk) => (body += chunk));
      req.on('end', () => {
        try {
          resolve(body ? JSON.parse(body) : {});
        } catch {
          resolve({ raw: body });
        }
      });
    });
  }

  private validateAuth(auth: AuthConfig, req: http.IncomingMessage): boolean {
    if (auth.type === 'none') return true;

    const authHeader =
      req.headers.authorization ||
      req.headers[auth.header?.toLowerCase() || 'authorization'];

    if (!authHeader) return false;

    const authHeaderStr = Array.isArray(authHeader)
      ? authHeader[0]
      : authHeader;

    if (!authHeaderStr) return false;

    switch (auth.type) {
      case 'jwt':
        return this.validateJWT(authHeaderStr, auth);
      case 'bearer':
        return authHeaderStr === `Bearer ${auth.token}`;
      case 'apikey':
        return authHeaderStr === auth.token;
      case 'basic':
        return authHeaderStr.startsWith('Basic ');
      case 'custom':
        const expectedValue = auth.prefix
          ? `${auth.prefix} ${auth.token}`
          : auth.token;
        return authHeaderStr === expectedValue;
      default:
        return false;
    }
  }

  private validateJWT(authHeader: string, auth: AuthConfig): boolean {
    try {
      const token = authHeader.replace('Bearer ', '');
      const decoded = jwt.verify(token, auth.secret || 'default-secret', {
        algorithms: [(auth.algorithm as any) || 'HS256'],
      });
      return true;
    } catch {
      return false;
    }
  }

  private validateRequest(
    validation: ValidationRule[],
    requestData: any,
  ): string | null {
    for (const rule of validation) {
      const fieldValue = this.getNestedField(requestData, rule.field);

      switch (rule.type) {
        case 'required':
          if (
            fieldValue === undefined ||
            fieldValue === null ||
            fieldValue === ''
          ) {
            return rule.message;
          }
          break;
        case 'email':
          if (fieldValue && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(fieldValue)) {
            return rule.message;
          }
          break;
        case 'number':
          if (fieldValue && isNaN(Number(fieldValue))) {
            return rule.message;
          }
          break;
        case 'minLength':
          if (fieldValue && fieldValue.length < (rule.value || 0)) {
            return rule.message;
          }
          break;
        case 'maxLength':
          if (fieldValue && fieldValue.length > (rule.value || 0)) {
            return rule.message;
          }
          break;
        case 'regex':
          if (
            fieldValue &&
            rule.value &&
            !new RegExp(rule.value.toString()).test(fieldValue)
          ) {
            return rule.message;
          }
          break;
      }
    }
    return null;
  }

  private getNestedField(obj: any, path: string): any {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }

  private evaluateConditionalResponses(
    conditions: any[],
    requestData: any,
  ): any | null {
    for (const condition of conditions) {
      try {
        // Simple condition evaluation (in production, use a safe evaluator)
        const conditionFunction = new Function(
          'request',
          `return ${condition.condition}`,
        );
        if (conditionFunction(requestData)) {
          return condition;
        }
      } catch (error) {
        this.log(
          `Error evaluating condition: ${condition.condition} - ${error}`,
        );
      }
    }
    return null;
  }

  private processDynamicVariables(response: string): string {
    return response
      .replace(/\{\{timestamp\}\}/g, new Date().toISOString())
      .replace(/\{\{uuid\}\}/g, this.generateUUID())
      .replace(/\{\{random\}\}/g, Math.random().toString())
      .replace(/\{\{faker\.name\}\}/g, this.generateFakeName())
      .replace(/\{\{faker\.email\}\}/g, this.generateFakeEmail())
      .replace(/\{\{increment\}\}/g, Date.now().toString());
  }

  private generateUUID(): string {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = (Math.random() * 16) | 0;
      const v = c === 'x' ? r : (r & 0x3) | 0x8;
      return v.toString(16);
    });
  }

  private generateFakeName(): string {
    const names = [
      'John Doe',
      'Jane Smith',
      'Bob Johnson',
      'Alice Williams',
      'Charlie Brown',
    ];
    return names[Math.floor(Math.random() * names.length)] || 'Unknown User';
  }

  private generateFakeEmail(): string {
    const domains = ['example.com', 'test.org', 'demo.net'];
    const name = this.generateFakeName().toLowerCase().replace(' ', '.');
    const domain = domains[Math.floor(Math.random() * domains.length)];
    return `${name}@${domain}`;
  }

  // log(message: string): void {
  //   const timestamp = new Date().toISOString();
  //   const logMessage = `[${timestamp}] ${message}`;

  //   console.log(logMessage);

  //   // Broadcast to WebSocket clients
  //   this.logClients.forEach((ws) => {
  //     if (ws.readyState === WebSocket.OPEN) {
  //       ws.send(logMessage);
  //     }
  //   });
  // }
  log(message: string): void {
    const timestamp = new Date().toISOString();
    const logMessage = `[${timestamp}] ${message}`;
    console.log(logMessage);

    // Only broadcast if not shutting down
    if (!this.isShuttingDown) {
      this.logClients.forEach((ws) => {
        if (ws.readyState === WebSocket.OPEN) {
          try {
            ws.send(logMessage);
          } catch (error) {
            // Ignore send errors during shutdown
          }
        }
      });
    }
  }

  updateConfig(newConfig: ServerConfig): void {
    this.config = newConfig;
    this.log('Server configuration updated');
  }

  // getStatus(): any {
  //   return {
  //     running: this.httpServer !== null,
  //     port: this.config.port,
  //     endpoints: this.config.endpoints.length,
  //     protocols: [...new Set(this.config.endpoints.map((ep) => ep.protocol))],
  //     timestamp: new Date().toISOString(),
  //     services: {
  //       http: this.httpServer !== null,
  //       https: this.httpsServer !== null,
  //       websocket: this.wsServer !== null,
  //       tcp: this.tcpServers.size > 0,
  //       udp: this.udpSockets.size > 0,
  //       mqtt: this.mqttClients.size > 0,
  //     },
  //   };
  // }

  // TCP Connection Handler
  getStatus(): any {
    return {
      running: this.httpServer !== null && !this.isShuttingDown,
      shuttingDown: this.isShuttingDown,
      port: this.config.port,
      endpoints: this.config.endpoints.length,
      protocols: [...new Set(this.config.endpoints.map((ep) => ep.protocol))],
      timestamp: new Date().toISOString(),
      connections: {
        http: this.httpConnections.size,
        https: this.httpsConnections.size,
        websocket: this.wsConnections.size,
        tcp: Array.from(this.tcpConnections.values()).reduce(
          (sum, set) => sum + set.size,
          0,
        ),
        logClients: this.logClients.length,
      },
      services: {
        http: this.httpServer !== null,
        https: this.httpsServer !== null,
        websocket: this.wsServer !== null,
        tcp: this.tcpServers.size > 0,
        udp: this.udpSockets.size > 0,
        mqtt: this.mqttClients.size > 0,
      },
    };
  }

  private handleTcpConnection(socket: net.Socket, endpoint: EndpointConfig) {
    this.log(`TCP connection established for endpoint ${endpoint.id}`);

    socket.on('data', async (data) => {
      try {
        // Simulate latency
        if (endpoint.latency > 0) {
          await new Promise((resolve) => setTimeout(resolve, endpoint.latency));
        }

        // Process the data and generate response
        let response = this.processDynamicVariables(endpoint.response);

        // Apply encoding
        const encoding = endpoint.protocolConfig?.tcp?.encoding || 'utf8';
        const responseBuffer = Buffer.from(
          response,
          encoding as BufferEncoding,
        );

        socket.write(responseBuffer);
        this.log(`TCP response sent for endpoint ${endpoint.id}`);
      } catch (error) {
        this.log(`TCP error for endpoint ${endpoint.id}: ${error}`);
        socket.destroy();
      }
    });

    socket.on('error', (error) => {
      this.log(`TCP socket error for endpoint ${endpoint.id}: ${error}`);
    });

    socket.on('close', () => {
      this.log(`TCP connection closed for endpoint ${endpoint.id}`);
    });
  }

  // UDP Message Handler
  private handleUdpMessage(
    msg: Buffer,
    rinfo: dgram.RemoteInfo,
    endpoint: EndpointConfig,
    socket: dgram.Socket,
  ) {
    this.log(
      `UDP message received for endpoint ${endpoint.id} from ${rinfo.address}:${rinfo.port}`,
    );

    setTimeout(async () => {
      try {
        // Process the message and generate response
        let response = this.processDynamicVariables(endpoint.response);
        const responseBuffer = Buffer.from(response, 'utf8');

        socket.send(responseBuffer, rinfo.port, rinfo.address, (error) => {
          if (error) {
            this.log(`UDP send error for endpoint ${endpoint.id}: ${error}`);
          } else {
            this.log(
              `UDP response sent for endpoint ${endpoint.id} to ${rinfo.address}:${rinfo.port}`,
            );
          }
        });
      } catch (error) {
        this.log(`UDP processing error for endpoint ${endpoint.id}: ${error}`);
      }
    }, endpoint.latency || 0);
  }

  // MQTT Simulation Handler
  private initializeMqttSimulation(endpoints: EndpointConfig[]) {
    for (const endpoint of endpoints) {
      const topic = endpoint.protocolConfig?.mqtt?.topic || endpoint.path;
      this.log(`MQTT simulation initialized for topic: ${topic}`);

      // Simulate MQTT publish/subscribe behavior
      // In a real implementation, you would connect to an actual MQTT broker
      const mqttClient = {
        topic,
        qos: endpoint.protocolConfig?.mqtt?.qos || 0,
        retain: endpoint.protocolConfig?.mqtt?.retain || false,
        clientId:
          endpoint.protocolConfig?.mqtt?.clientId || `sim_${endpoint.id}`,

        publish: (message: string) => {
          this.log(`MQTT publish to ${topic}: ${message}`);
          // Simulate message delivery
          setTimeout(() => {
            const response = this.processDynamicVariables(endpoint.response);
            this.log(`MQTT message processed for ${topic}: ${response}`);
          }, endpoint.latency || 0);
        },
      };

      this.mqttClients.set(endpoint.id, mqttClient);
    }
  }

  // Initialize Other Protocols
  private async initializeOtherProtocols(
    endpoints: EndpointConfig[],
    basePort: number,
  ) {
    const protocols = [...new Set(endpoints.map((ep) => ep.protocol))];

    for (const protocol of protocols) {
      switch (protocol) {
        case 'gRPC':
          this.log('gRPC protocol detected - initializing gRPC simulation');
          // In a real implementation, initialize gRPC server
          break;

        case 'GraphQL':
          this.log('GraphQL protocol detected - schema validation enabled');
          // GraphQL is handled by HTTP server with special routing
          break;

        case 'AMQP':
          this.log(
            'AMQP protocol detected - initializing message queue simulation',
          );
          // In a real implementation, connect to RabbitMQ or similar
          break;

        case 'Redis PubSub':
          this.log(
            'Redis PubSub protocol detected - initializing Redis simulation',
          );
          // In a real implementation, connect to Redis
          break;

        case 'Server-Sent Events (SSE)':
          this.log('SSE protocol detected - enabling event streaming');
          // SSE is handled by HTTP server with special headers
          break;

        case 'FTP':
        case 'SFTP':
          this.log(
            `${protocol} protocol detected - initializing file transfer simulation`,
          );
          // In a real implementation, start FTP server
          break;

        case 'SMTP':
          this.log('SMTP protocol detected - initializing email simulation');
          // In a real implementation, start SMTP server
          break;

        default:
          if (
            ![
              'HTTP',
              'HTTPS',
              'WebSocket',
              'WebSocket Secure (WSS)',
              'TCP',
              'UDP',
              'MQTT',
            ].includes(protocol)
          ) {
            this.log(
              `Custom protocol detected: ${protocol} - using generic handler`,
            );
          }
          break;
      }
    }
  }

  // Enhanced stop method for multiple protocols
  // async stop(): Promise<void> {
  //   this.log('Stopping multi-protocol server...');

  //   // Stop HTTP servers
  //   if (this.httpServer) {
  //     await new Promise<void>((resolve) =>
  //       this.httpServer!.close(() => resolve()),
  //     );
  //     this.httpServer = null;
  //   }

  //   if (this.httpsServer) {
  //     await new Promise<void>((resolve) =>
  //       this.httpsServer!.close(() => resolve()),
  //     );
  //     this.httpsServer = null;
  //   }

  //   // Stop WebSocket servers
  //   if (this.wsServer) {
  //     this.wsServer.close();
  //     this.wsServer = null;
  //   }

  //   if (this.wssServer) {
  //     this.wssServer.close();
  //     this.wssServer = null;
  //   }

  //   // Stop TCP servers
  //   for (const [id, server] of this.tcpServers) {
  //     server.close();
  //     this.log(`TCP server stopped for endpoint ${id}`);
  //   }
  //   this.tcpServers.clear();

  //   // Stop UDP sockets
  //   for (const [id, socket] of this.udpSockets) {
  //     socket.close();
  //     this.log(`UDP socket closed for endpoint ${id}`);
  //   }
  //   this.udpSockets.clear();

  //   // Clear MQTT clients
  //   this.mqttClients.clear();

  //   // Stop log WebSocket server
  //   if (this.logWSS) {
  //     this.logWSS.close();
  //     this.logWSS = null;
  //     this.logClients = [];
  //   }

  //   // Clear other resources
  //   this.servers = [];
  //   this.rateLimiters.clear();
  //   this.requestCounts.clear();
  //   this.sseClients.clear();

  //   this.log('Multi-protocol server stopped successfully');
  // }
  async stop(): Promise<void> {
    if (this.isShuttingDown) {
      this.log('Shutdown already in progress...');
      return;
    }

    this.isShuttingDown = true;
    this.log('🛑 Stopping multi-protocol server...');

    try {
      // 🔑 STEP 1: Close all WebSocket connections
      this.log(`Closing ${this.wsConnections.size} WebSocket connections...`);
      for (const ws of this.wsConnections) {
        try {
          ws.terminate(); // Force close, don't wait for handshake
        } catch (error) {
          this.log(`Error closing WebSocket: ${error}`);
        }
      }
      this.wsConnections.clear();

      // 🔑 STEP 2: Close WebSocket servers
      if (this.wsServer) {
        await this.closeWebSocketServer(this.wsServer, 'WebSocket');
        this.wsServer = null;
      }

      if (this.wssServer) {
        await this.closeWebSocketServer(this.wssServer, 'WebSocket Secure');
        this.wssServer = null;
      }

      // 🔑 STEP 3: Close log WebSocket server
      if (this.logWSS) {
        this.log(`Closing ${this.logClients.length} log WebSocket clients...`);
        this.logClients.forEach((ws) => {
          try {
            ws.terminate();
          } catch (error) {
            this.log(`Error closing log client: ${error}`);
          }
        });

        await this.closeWebSocketServer(this.logWSS, 'Log WebSocket');
        this.logWSS = null;
        this.logClients = [];
      }

      // 🔑 STEP 4: Close SSE connections
      this.log(`Closing ${this.sseClients.size} SSE connections...`);
      for (const [id, responses] of this.sseClients) {
        responses.forEach((res) => {
          try {
            res.end();
          } catch (error) {
            this.log(`Error closing SSE connection ${id}: ${error}`);
          }
        });
      }
      this.sseClients.clear();

      // 🔑 STEP 5: Stop HTTP server with timeout
      if (this.httpServer) {
        this.log(
          `Closing HTTP server with ${this.httpConnections.size} active connections...`,
        );

        await this.closeServerWithTimeout(
          this.httpServer,
          this.httpConnections,
          'HTTP',
          5000,
        );

        this.httpServer = null;
      }

      // 🔑 STEP 6: Stop HTTPS server with timeout
      if (this.httpsServer) {
        this.log(
          `Closing HTTPS server with ${this.httpsConnections.size} active connections...`,
        );

        await this.closeServerWithTimeout(
          this.httpsServer,
          this.httpsConnections,
          'HTTPS',
          5000,
        );

        this.httpsServer = null;
      }

      // 🔑 STEP 7: Stop TCP servers
      for (const [id, server] of this.tcpServers) {
        const connections = this.tcpConnections.get(id);

        if (connections && connections.size > 0) {
          this.log(
            `Closing ${connections.size} TCP connections for endpoint ${id}...`,
          );
          this.forceCloseConnections(connections);
        }

        await this.closeTcpServer(server, id);
      }
      this.tcpServers.clear();
      this.tcpConnections.clear();

      // 🔑 STEP 8: Stop UDP sockets
      for (const [id, socket] of this.udpSockets) {
        await this.closeUdpSocket(socket, id);
      }
      this.udpSockets.clear();

      // 🔑 STEP 9: Disconnect MQTT clients
      for (const [id, client] of this.mqttClients) {
        try {
          if (client && typeof client.end === 'function') {
            await new Promise<void>((resolve) => {
              client.end(true, () => resolve()); // Force disconnect
              setTimeout(resolve, 1000); // Timeout after 1s
            });
            this.log(`MQTT client disconnected for endpoint ${id}`);
          }
        } catch (error) {
          this.log(`Error disconnecting MQTT client ${id}: ${error}`);
        }
      }
      this.mqttClients.clear();

      // 🔑 STEP 10: Clear other resources
      this.servers = [];
      this.rateLimiters.clear();
      this.requestCounts.clear();

      this.log('✅ Multi-protocol server stopped successfully');
    } catch (error) {
      this.log(`❌ Error during server shutdown: ${error}`);
      throw error;
    } finally {
      this.isShuttingDown = false;
    }
  }

  /**
   * Close server with timeout and forced connection cleanup
   */
  private async closeServerWithTimeout(
    server: http.Server | https.Server,
    connections: Set<net.Socket>,
    name: string,
    timeout: number = 5000,
  ): Promise<void> {
    return new Promise((resolve) => {
      const timeoutId = setTimeout(() => {
        this.log(`${name} server close timeout, forcing connection closure...`);
        this.forceCloseConnections(connections);

        // Force close the server
        server.close(() => {
          this.log(`${name} server closed after forcing connections`);
          resolve();
        });

        // Resolve anyway after additional timeout
        setTimeout(resolve, 1000);
      }, timeout);

      server.close(() => {
        clearTimeout(timeoutId);
        this.log(`${name} server closed gracefully`);
        resolve();
      });

      // Also try to force close connections immediately
      this.forceCloseConnections(connections);
    });
  }

  /**
   * Force close all connections in a set
   */
  private forceCloseConnections(connections: Set<net.Socket>) {
    for (const socket of connections) {
      try {
        socket.destroy();
      } catch (error) {
        // Ignore errors, socket might already be closed
      }
    }
    connections.clear();
  }

  /**
   * Close WebSocket server safely
   */
  private async closeWebSocketServer(
    wss: WebSocket.Server,
    name: string,
  ): Promise<void> {
    return new Promise<void>((resolve) => {
      try {
        // Terminate all client connections first
        if (wss.clients) {
          wss.clients.forEach((client: any) => {
            try {
              client.terminate();
            } catch (error) {
              // Ignore
            }
          });
        }

        // Close the server
        wss.close(() => {
          this.log(`${name} server closed`);
          resolve();
        });

        // Force resolve after timeout
        setTimeout(() => {
          this.log(`${name} server close forced after timeout`);
          resolve();
        }, 2000);
      } catch (error) {
        this.log(`Error closing ${name} server: ${error}`);
        resolve();
      }
    });
  }

  /**
   * Close TCP server safely
   */
  private async closeTcpServer(server: net.Server, id: string): Promise<void> {
    return new Promise<void>((resolve) => {
      const timeout = setTimeout(() => {
        this.log(`TCP server ${id} close timeout`);
        resolve();
      }, 3000);

      server.close((err) => {
        clearTimeout(timeout);
        if (err) {
          this.log(`TCP server ${id} close error: ${err.message}`);
        } else {
          this.log(`TCP server stopped for endpoint ${id}`);
        }
        resolve();
      });
    });
  }

  /**
   * Close UDP socket safely
   */
  private async closeUdpSocket(
    socket: dgram.Socket,
    id: string,
  ): Promise<void> {
    return new Promise<void>((resolve) => {
      try {
        const timeout = setTimeout(() => {
          this.log(`UDP socket ${id} close timeout`);
          resolve();
        }, 2000);

        socket.close(() => {
          clearTimeout(timeout);
          this.log(`UDP socket closed for endpoint ${id}`);
          resolve();
        });
      } catch (error) {
        this.log(`Error closing UDP socket ${id}: ${error}`);
        resolve();
      }
    });
  }

  /**
   * Graceful shutdown handler
   */
  async gracefulShutdown(signal: string): Promise<void> {
    this.log(`Received ${signal} signal, starting graceful shutdown...`);

    try {
      // Give active requests 2 seconds to complete
      if (this.httpConnections.size > 0 || this.wsConnections.size > 0) {
        this.log('Waiting 2 seconds for active requests to complete...');
        await new Promise((resolve) => setTimeout(resolve, 2000));
      }

      await this.stop();
      this.log('Graceful shutdown completed');
      process.exit(0);
    } catch (error) {
      this.log(`Error during graceful shutdown: ${error}`);
      process.exit(1);
    }
  }
}

export default SimulatedServerService;
