<script lang="ts">
    import { onMount, setContext } from "svelte";
    
    import {
        Button,
        Textarea,
        Select,
        Radio,
        RadioGroup,
        Checkbox,
    } from "deskblocks";
    import { DB, initApp } from "../lib/util";

    let App: APP;

    setContext("App", App);

    let domain: string;
    let views: string[] = [];
    let agents: string[] = [];
    let selectedView: string;
    let selectedAgents: string[];
    let selectedDepartment: string;
    let statusType: string;
    let revokeBlueprint = false;
    let selectedValues: string[] = [];
    let selectedAgentsTest: string[] = ["189902000110338445","189902000110338371","189902000053986019"];

    let hasPermission: boolean | null = null;

    async function getDepartment() {
        await ZOHODESK.get("department.id")
            .then(async function (departmentResponse) {
                selectedDepartment = departmentResponse["department.id"];
            })
            .catch(function (error) {
                console.error("Error fetching department:", error);
            });
    }

    async function getViews() {
        let payload = {
            url: getRequestUrl(
                `api/v1/views?module=tickets&departmentId=${selectedDepartment}`,
            ),
            method: "GET",
            headers: {},
            connectionLinkName: "sarantest",
            responseType: "json",
            data: {},
        };
        ZOHODESK.request(payload).then(async (res) => {
            views = res.data.statusMessage.data.map((view) => ({
                value: view.id,
                label: view.name,
            }));
        });
    }

    async function getAgents() {
        let payload: RequestOptions = {
            url: getRequestUrl(
                `api/v1/agents`,
            ),
            type: "GET",
            headers: {},
            connectionLinkName: "sarantest",
            responseType: "json",
            data: {
                departmentId: selectedDepartment,
                status:"ACTIVE",
                isConfirmed:true,
                rolePermissionType:"${NON_LIGHT}",
                from:1,
                limit:200
            },
        };
        ZOHODESK.request(payload).then(async (res) => {
            agents = res.data.statusMessage.data.map((agent) => ({
                value: agent.id,
                label: agent.name,
            }));
        });
    }
    async function executeTicketAssignment() {
        let allTicketIds = await getAllTicketIds(
            selectedView[0].value,
            selectedDepartment,
            statusType[0].value,
        );
        assignTicketsToAgents(allTicketIds, selectedAgents.map((agent) => agent.value));


    }

    async function revokeBlueprints(ticketIds: string[]) {
        let payload: RequestOptions = {
            url: getRequestUrl("api/v1/blueprints/revokeMany"),
            type: "POST",
            headers: {},
            connectionLinkName: "sarantest",
            postBody: {
                ids: ticketIds,
            },
            data: {},
        };

        await ZOHODESK.request(payload);
    }

    async function assignTicketsToAgents(
        allTicketIds: string[],
        agentIds: string[],
        revokeBlueprint: boolean = false,
    ) {
        let totalTickets = allTicketIds.length;
        let totalAgents = agentIds.length;
        let balanceTickets = totalTickets % totalAgents;
        let ticketsPerAgent = Math.floor(totalTickets / totalAgents);
        let iteration = 0;
        console.log("totalTickets", totalTickets);
        console.log("totalAgents", totalAgents);
        console.log("balanceTickets", balanceTickets);
        console.log("ticketsPerAgent", ticketsPerAgent);
        agentIds.forEach((agentId) => {
            let from = iteration * ticketsPerAgent;
            let limit = (iteration + 1) * ticketsPerAgent;
            if (balanceTickets > iteration) {
                limit += 1;
            }
            let ticketIds = allTicketIds.slice(from, limit);

            if (revokeBlueprint) {
                revokeBlueprints(ticketIds);
            }
            updateTicketAssignee(ticketIds, agentId.toString());

            iteration++;
        });
    }

    async function getFieldByApiName(apiName: string) {
        let payload: RequestOptions = {
            url: getRequestUrl(`api/v1/organizationFields`),
            type: "GET",
            headers: {},
            connectionLinkName: "sarantest",
            responseType: "json",
            data: {
                module: "tickets",
                departmentId: selectedDepartment,
                apiNames: apiName,
            },
        };
        return ZOHODESK.request(payload).then((res) => {
            let field = res.data.statusMessage.data[0];
            return field;
        });
    }

    async function getTicketCount(viewId: string, departmentId: string) {
        let payload: RequestOptions = {
            url: getRequestUrl(
                `api/v1/tickets/count?viewId=${viewId}&departmentId=${departmentId}`,
            ),
            type: "GET",
            headers: {},
            connectionLinkName: "sarantest",
            responseType: "json",
            data: {},
        };
        return ZOHODESK.request(payload).then((res) => {
            return res.data.statusMessage.count;
        });
    }

    async function getAllTicketIds(
        viewId: string,
        departmentId: string,
        statusType: string,
    ) {
        let allTicketIds = [];
        let from = 1;
        const limit = 100;
        let statuses = [];
        let fieldConditions = [];
        if (statusType !== "ALL") {
            statuses = await getStatusList(statusType);
            fieldConditions.push(getFieldCriteria("status", statuses, "is"));
        }

        let criteria = {
            criteria: { fieldConditions },
        };
        console.log("criteria", criteria);
        while (true) {
            let requestPayload: RequestOptions = {
                url: getRequestUrl(`api/v1/tickets`),
                type: "GET",
                headers: {
                    "Content-Type": "application/json",
                },
                connectionLinkName: "sarantest",
                responseType: "json",
                data: {
                    filters: criteria,
                    viewId: viewId,
                    departmentId: departmentId,
                    from: from,
                    limit: limit,
                    fields : "id",
                },
            };

            const response = await ZOHODESK.request(requestPayload);
            let ticketsCount = response.data.statusMessage.data.length;
            if (ticketsCount > 0) {
                allTicketIds = allTicketIds.concat(
                    response.data.statusMessage.data.map((ticket) => ticket.id),
                );
            }

            if (ticketsCount < limit) {
                break;
            }
            from += limit;
        }
        console.log("allTicketIds", allTicketIds.length);
        return allTicketIds;
    }

    async function updateTicketAssignee(
        allTicketIds: string[],
        agentId: string,
    ) {
        const batchSize = 50;
        for (let i = 0; i < allTicketIds.length; i += batchSize) {
            const batchIds = allTicketIds.slice(i, i + batchSize);

            let updatePayload: RequestOptions = {
                url: getRequestUrl("api/v1/tickets/updateMany"),
                type: "POST",
                headers: {},
                connectionLinkName: "sarantest",
                postBody: {
                    ids: batchIds,
                    fieldName: "assigneeId",
                    fieldValue: agentId,
                },
                data: {},
            };

            await ZOHODESK.request(updatePayload).then(async (res) => {
                console.log("res", res);
            });
        }
    }

    function getFieldCriteria(
        fieldApiName: string,
        value: any,
        condition: string,
    ) {
        return {
            field: fieldApiName,
            data: value,
            con: condition,
        };
    }

    async function getStatusList(statusType: string) {
        let field = await getFieldByApiName("status");
        console.log("statusType", statusType);
        let statuses = field.allowedValues
            .filter((v) => v.statusType === statusType)
            .map((v) => v.value);

        console.log("statuses  111", statuses);
        return statuses;
    }

    function getRequestUrl(pathUrl: string) {
        return `${domain}/${pathUrl}`;
    }

    function handleSelection() {
        console.log("Selected Agents:", selectedAgents);
    }

    function toggleAgent(agentId) {
        if (selectedAgents.includes(agentId)) {
            selectedAgents = selectedAgents.filter((id) => id !== agentId);
        } else {
            selectedAgents = [...selectedAgents, agentId];
        }
        handleSelection();
    }

    async function getMyProfilePermissions() {
        let payload: RequestOptions = {
            url: getRequestUrl("api/v1/myProfilePermissions"),      
            type: "GET",
            headers: {},
            connectionLinkName: "sarantest",
            responseType: "json",
            data: {},
        };
        return ZOHODESK.request(payload).then((res) => {
            return res.data.statusMessage.permissions;
        });
    }  
    async function validateTicketAssignmentPermission() {
        try {
            let permissions = await getMyProfilePermissions();
            let ticketPermission = permissions.tickets;
            console.log("ticketPermission", permissions); 
            return ticketPermission.edit;
        } catch (error) {
            console.error("Error checking permissions:", error);
            // Return false if there's an error, assuming no permission
            return false;
        }
    } 
    const statusOptions = [
        { label: 'All', value: 'ALL',preselected:true },
		{ label: 'Open', value: 'OPEN' },
		{ label: 'On Hold', value: 'ON_HOLD' },
		{ label: 'Closed', value: 'CLOSED' }
	];
    onMount(async () => {
        App = await initApp();
        domain = App.meta.deskDomainUrl;
        hasPermission = await validateTicketAssignmentPermission();
        await getDepartment();
        await getViews();
        await getAgents();
        
        
        // Check permissions after domain is set
       
    });
   
</script>

<main>
    {#if hasPermission === null}
        <p>Checking permissions...</p>
    {:else if hasPermission} 
        <Select 
        bind:selected={selectedView} 
        options={views} 
        placeholder="Choose a view" 
        resetFilterOnAdd={true}
        />
      
        <Select 
        bind:selected={selectedAgents} 
        options={agents} 
        placeholder="Choose a Agent" 
        maxSelect={null}
        resetFilterOnAdd={true}
        />
        <Select 
        bind:selected={statusType} 
        options={statusOptions} 
        placeholder="Choose a Status" 
        
        />

      
        <Checkbox bind:checked={revokeBlueprint}>Revoke Blueprints</Checkbox>
        <Button disabled={!hasPermission} on:click={executeTicketAssignment}>Execute Ticket Assignment</Button>
    {:else}
        <p>You don't have permission to perform this action.</p>
    {/if}
</main>
