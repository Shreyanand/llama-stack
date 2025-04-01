## RFC: Update LlamaStack MCP Client to Support Resources and Prompts

**Author(s):** Shreyanand

**Date:** 2025-04-01  

**Status:** Proposed  

---

## Summary

This RFC proposes extending LlamaStack's MCP (Model Context Protocol) client to support two additional MCP primitives: Resources and Prompts, alongside existing support for MCP Tools. Additionally, it proposes renaming the existing `/toolgroups` endpoint to a generalized `/mcp` endpoint to unify management of MCP servers, tools, resources, and prompts. Implementing these changes will enable LlamaStack to fully leverage advanced MCP servers and significantly enhance its integration capabilities.

## Motivation

MCP provides a standardized approach for integrating external APIs and contextual data into LLM applications. MCP consists of three key [primitives](https://github.com/modelcontextprotocol/python-sdk?tab=readme-ov-file#mcp-primitives):

1. **[Tools](https://spec.modelcontextprotocol.io/specification/2025-03-26/server/tools/)**: Already implemented by LlamaStack and can be accessed through the Agents API.
2. **[Resources](https://spec.modelcontextprotocol.io/specification/2025-03-26/server/resources/)**: (To be implemented) Application-controlled contextual data (e.g., API responses for validation, files, database records).
3. **[Prompts](https://spec.modelcontextprotocol.io/specification/2025-03-26/server/prompts/)**: (To be implemented) User-controlled interactive templates provided by servers for specific tasks or formatting (e.g., prompt for summarizing GitHub PRs/issues provided by github MCP server).

Currently, LlamaStack implements only the MCP Tools primitive, limiting its capability to fully integrate and leverage advanced MCP servers, such as those managing resources and custom prompts (e.g., [Docker MCP Server](https://github.com/ckreiling/mcp-server-docker/blob/main/src/mcp_server_docker/server.py)).

### Positive Impact:
- Enhanced connectivity with emerging MCP servers.
- Foundation to add upcoming MCP features such as authentication and sampling.

## Proposed Solution

We propose adding a new `v1/mcp` endpoint, modeled similarly to the existing ToolGroups endpoint, potentially replacing it. This new endpoint will manage MCP servers, synchronizing their supported tools, resources, and prompts with the LlamaStack server.

```
@runtime_checkable
@trace_protocol
class MCPServers(Protocol):
    @webmethod(route="/mcp", method="POST")
    async def register_server(
        self,
        mcp_id: str,
        provider_id: str,
        mcp_endpoint: URL,
        args: Optional[Dict[str, Any]] = None,
    ) -> None:
        """
        Register an MCP server.
        - If the server already exists, synchronize its resources, prompts, and tools.
        - If it doesn't exist, fetch and store all tools, prompts, and resources from the MCP server.
        """
        ...

    @webmethod(route="/mcp/{mcp_id:path}", method="GET")
    async def get_server(
        self,
        mcp_id: str,
    ) -> MCPServer:
        """
        Retrieve details about the registered MCP server.
        """
        ...

    @webmethod(route="/mcp", method="GET")
    async def list_servers(self) -> ListMCPServersResponse:
        """
        List all registered MCP servers.
        """
        ...

    @webmethod(route="/mcp/{mcp_id:path}", method="DELETE")
    async def unregister_server(
        self,
        mcp_id: str,
    ) -> None:
        """
        Unregister an MCP server and remove associated data.
        """
        ...

    @webmethod(route="/tools/{tool_name:path}", method="GET")
    async def get_tool(
        self,
        tool_name: str,
    ) -> Tool:
        """
        Retrieve details of a specific MCP tool by name.
        """
        ...

    @webmethod(route="/tools", method="GET")
    async def list_tools(
        self,
        mcp_id: Optional[str] = None,
    ) -> ListToolsResponse:
        """
        List tools optionally filtered by MCP server ID.
        """
        ...

    @webmethod(route="/prompts/{prompt_name:path}", method="GET")
    async def get_prompt(
        self,
        prompt_name: str,
    ) -> Prompt:
        """
        Retrieve details of a specific prompt.
        """
        ...

    @webmethod(route="/prompts", method="GET")
    async def list_prompts(
        self,
        mcp_id: Optional[str] = None,
    ) -> ListPromptsResponse:
        """
        List prompts optionally filtered by MCP server ID.
        """
        ...

    @webmethod(route="/mcp-resources/{resource_name:path}", method="GET")
    async def get_resource(
        self,
        resource_name: str,
    ) -> Resource:
        """
        Retrieve a specific resource managed by MCP.
        """
        ...

    @webmethod(route="/mcp-resources", method="GET")
    async def list_resources(
        self,
        mcp_id: Optional[str] = None,
    ) -> ListResourcesResponse:
        """
        List resources optionally filtered by MCP server ID.
        """
        ...
```
## Alternatives and Concerns

### Alternatives
An alternative to adding a new `/mcp` endpoint could be to to keep the current `/toolgroups` and register prompts and resources when a MCP server is added as a toolgroup. However, there are potential concerns with that approach:
* 1. Resources and prompts don't logically fall in a "toolgroup" and it will be confusing if we register MCP as a toolgroup and add endpoints for `/prompts` and `/resources `
* 2. Toolgroups only exist for MCP server tools: In a recent tools API rework, all other toolgroups will be converted to `types`: see [PR](https://github.com/meta-llama/llama-stack/pull/1764)

### Concerns
1. Why do we need prompts on the server side, can it not be on the client?

    If the remote MCP server is registered on the llamastack server, the builtin prompts will have to be surfaced through the llamastack server. The same argument follows for MCP server tools vs client tools.

2. The word "resources" can be confused with the existing `Resource` class in llamastack which includes models, benchmarks, datasets, etc. 

    The suggested API endpoint is `/mcp-resources` but it is open for discussion.


## Next Steps

1. Get feedback and validate
2. Design and define interfaces/endpoints for MCP registering, Resources and Prompts within LlamaStack MCP Client.
3. Implement changes in the [existing LlamaStack MCP Client implementation](https://github.com/meta-llama/llama-stack/blob/main/llama_stack/providers/remote/tool_runtime/model_context_protocol/model_context_protocol.py) 
4. Implement tests for the new features.

---
