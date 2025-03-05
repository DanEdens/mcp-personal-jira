#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Retrieve environment variables
const JIRA_INSTANCE_URL = process.env.JIRA_INSTANCE_URL;
const JIRA_API_KEY = process.env.JIRA_API_KEY;
const JIRA_USER_EMAIL = process.env.JIRA_USER_EMAIL;

// Validate environment variables
if (!JIRA_INSTANCE_URL || !JIRA_API_KEY || !JIRA_USER_EMAIL) {
  console.error(
    "Error: JIRA_INSTANCE_URL, JIRA_USER_EMAIL, and JIRA_API_KEY must be set in the environment.",
  );
  process.exit(1);
}

// Initialize the server
const server = new Server(
  {
    name: "jira-mcp",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  },
);

// Define available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "jql_search",
        description: "Perform enhanced JQL search in Jira",
        inputSchema: {
          type: "object",
          properties: {
            jql: { type: "string", description: "JQL query string" },
            nextPageToken: {
              type: "string",
              description: "Token for next page",
            },
            maxResults: {
              type: "integer",
              description: "Maximum results to fetch",
            },
            fields: {
              type: "array",
              items: { type: "string" },
              description: "List of fields to return for each issue",
            },
            expand: {
              type: "string",
              description: "Additional info to include in the response",
            },
          },
          required: ["jql"],
        },
      },
      {
        name: "get_issue",
        description: "Retrieve details about an issue by its ID or key.",
        inputSchema: {
          type: "object",
          properties: {
            issueIdOrKey: {
              type: "string",
              description: "ID or key of the issue",
            },
            fields: {
              type: "array",
              items: { type: "string" },
              description: "Fields to include in the response",
            },
            expand: {
              type: "string",
              description: "Additional information to include in the response",
            },
            properties: {
              type: "array",
              items: { type: "string" },
              description: "Properties to include in the response",
            },
            failFast: {
              type: "boolean",
              description: "Fail quickly on errors",
              default: false,
            },
          },
          required: ["issueIdOrKey"],
        },
      },
    ],
  };
});

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "jql_search") {
    const { jql, nextPageToken, maxResults, fields, expand } = args;
    try {
      const response = await fetch(`${JIRA_INSTANCE_URL}/rest/api/2/search`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Basic ${Buffer.from(`${JIRA_USER_EMAIL}:${JIRA_API_KEY}`).toString("base64")}`,
        },
        body: JSON.stringify({
          jql,
          startAt: nextPageToken || 0,
          maxResults: maxResults || 50,
          fields: fields || ["*all"],
          expand,
        }),
      });

      if (!response.ok) {
        throw new Error(`Jira API Error: ${response.statusText}`);
      }

      const data = await response.json();
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(data, null, 2), // Format JSON response
          },
        ],
      };
    } catch (error) {
      return {
        isError: true,
        content: [
          {
            type: "text",
            text: `Error: ${error.message}`,
          },
        ],
      };
    }
  } else if (name === "get_issue") {
    const { issueIdOrKey, fields, expand, properties, failFast } = args;
    try {
      const queryParams = new URLSearchParams();

      if (fields) queryParams.append("fields", fields.join(","));
      if (expand) queryParams.append("expand", expand);
      if (properties) queryParams.append("properties", properties.join(","));
      if (failFast !== undefined)
        queryParams.append("failFast", String(failFast));

      const response = await fetch(
        `${JIRA_INSTANCE_URL}/rest/api/2/issue/${issueIdOrKey}?${queryParams.toString()}`,
        {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Basic ${Buffer.from(`${JIRA_USER_EMAIL}:${JIRA_API_KEY}`).toString("base64")}`,
          },
        },
      );

      if (!response.ok) {
        throw new Error(`Jira API Error: ${response.statusText}`);
      }

      const data = await response.json();
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(data, null, 2), // Format JSON response
          },
        ],
      };
    } catch (error) {
      return {
        isError: true,
        content: [
          {
            type: "text",
            text: `Error: ${error.message}`,
          },
        ],
      };
    }
  }

  throw new Error(`Tool not found: ${name}`);
});

// Start the server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.log("JIRA MCP Server is running.");
}

main().catch((error) => {
  console.error("Error starting the server:", error);
});
