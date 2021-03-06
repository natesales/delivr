<script>
    import Button from "./Button.svelte";
    import TextInput from "./TextInput.svelte";
    import Dropdown from "./Dropdown.svelte";
    import {onMount} from "svelte";
    import NumberInput from "./NumberInput.svelte";
    import {addSnackbar} from '../utils'
    import ToggleButton from "./ToggleButton.svelte";
    import CheckBoxes from "./CheckBoxes.svelte";
    import {API} from "../stores";

    let showAddRecord = false;

    export let zone;
    let records;

    let nodes;

    let type, label, value, priority, port, weight;
    type = "A";
    let ttl = 86400;
    let proxied = false;
    let showNodePinning = false;

    let snackbarEnabled = false;
    let snackbarColor = "green";
    let snackbarMessage = "";
    let snackbarTitle = "";

    function getPinnedNodes() {
        const checked = [];
        const inputs = document.getElementsByTagName("input");
        for (let i = 0; i < inputs.length; i++) {
            if (inputs[i].type === "checkbox") {
                if (inputs[i].checked) {
                    checked.push(inputs[i].value)
                }
            }
        }

        return checked.join(", ")
    }

    function submitForm() {
        let _label;

        if (label === "@") {
            _label = zone + "."
        } else {
            _label = label + "." + zone + "."
        }

        let body = {
            type: type,
            label: _label,
            value: value,
            priority: priority,
            port: port,
            weight: weight,
            ttl: ttl,
            proxied: proxied
        }

        if (showNodePinning) {
            body["pinned_nodes"] = getPinnedNodes()
        }

        fetch($API + "zone/" + zone + "/add", {
            method: "POST",
            headers: {
                "Content-Type": "application/json",
            },
            credentials: "include",
            body: JSON.stringify(body)
        })
            .then((response) => response.json())
            .then((data) => addSnackbar("zone_add", data["message"], data["success"] ? "green" : "red"))
            .then(() => loadRecords());
    }

    function deleteRecord(index) {
        if (confirm("Are you sure you want to delete this record?")) {
            fetch($API + "zone/" + zone + "/delete_record/" + index, {
                method: "POST",
                credentials: "include"
            })
                .then(response => response.json())
                .then(data => addSnackbar("delete_record", data["message"], data["success"] ? "green" : "red"))
                .then(() => loadRecords());
        }
    }

    function loadRecords(nothing) {
        if (zone !== undefined) {
            fetch($API + "zone/" + zone + "/records", {
                credentials: "include"
            })
                .then(response => response.json())
                .then(data => {
                    if (data["success"]) {
                        records = data["message"];
                    } else {
                        addSnackbar("zone_records", data["message"], "red");
                    }
                });
        } else {
            console.log("Zone undefined, holding.")
        }
    }

    function exportRecords() {
        fetch($API + "zones/" + zone + "/export", {
            credentials: "include"
        })
            .then(response => response.json())
            .then(data => {
                if (data["success"]) {
                    addSnackbar("zone_export", "Downloading zone file", "green");
                    let hiddenElement = document.createElement('a');
                    hiddenElement.href = 'data:attachment/text,' + encodeURI(data["message"]);
                    hiddenElement.target = '_blank';
                    hiddenElement.download = "db." + zone;
                    hiddenElement.click();
                } else {
                    addSnackbar("zone_export", data["message"], "red");
                }
            });
    }

    function deleteZone() {
        if (confirm("Are you sure you want to delete this zone?")) {
            fetch($API + "zone/" + zone + "/delete", {
                credentials: "include",
                method: "POST"
            })
                .then(response => response.json())
                .then(data => addSnackbar("delete_record", data["message"], data["success"] ? "green" : "red"))
                .then(() => {
                    location.hash = "";
                    location.reload();
                })
        }
    }

    function loadNodes() {
        fetch($API + "nodes/list", {
            credentials: "include"
        })
            .then(response => response.json())
            .then(data => {
                nodes = data["message"]
            })
    }

    $:loadRecords(zone);
    $: {
        if (zone !== undefined && zone.endsWith("arpa")) {
            console.log("Reverse zone")
            type = "PTR"
        } else {
            console.log("Forward zone")
            type = "A"
        }
    }

    onMount(() => {
        loadRecords();
        loadNodes();
    });
</script>

<main>
    <div class="header-container">
        <div>
            <h2>Records</h2>
            <div>
                <span style="padding-top: 1px">
                    <Button color="red" icon="delete" onclick={() => deleteZone()} size="1.25rem"/>
                </span>

                <span>
                    <Button icon="get_app" onclick={() => exportRecords()} size="1rem">Export</Button>
                </span>

                <span>
                    <Button icon="add" inverted=true onclick={() => {showAddRecord = !showAddRecord}} size="1rem">Add Record</Button>
                </span>
            </div>
        </div>

        {#if showAddRecord}
            <div class="record-add-container">
                <div class="record-add-element-select">
                    <Dropdown id="add-type" bind:content={type} small>
                        {#if zone && zone.endsWith("arpa")}
                            <option value="PTR">PTR</option>
                        {:else}
                            <option value="A">A</option>
                            <option value="AAAA">AAAA</option>
                            <option value="CNAME">CNAME</option>
                            <option value="TXT">TXT</option>
                            <option value="MX">MX</option>
                            <option value="SRV">SRV</option>
                        {/if}
                    </Dropdown>
                </div>

                {#if proxied}
                    <div class="record-add-element-number">
                        <NumberInput placeholder="Auto" id="add-value" disabled/>
                    </div>
                {:else}
                    <div class="record-add-element-number">
                        <NumberInput placeholder="TTL" id="add-value" bind:content={ttl}/>
                    </div>
                {/if}

                <div class="record-add-element">
                    <TextInput placeholder="Label" id="add-label" bind:content={label}/>
                </div>

                {#if type === "MX" || type === "SRV" }
                    <div class="record-add-element-number">
                        <NumberInput placeholder="Priority" id="add-value" bind:content={priority}/>
                    </div>
                {/if}

                {#if type === "SRV" }
                    <div class="record-add-element-number">
                        <NumberInput placeholder="Weight" id="add-value" bind:content={weight}/>
                    </div>
                    <div class="record-add-element-number">
                        <NumberInput placeholder="Port" id="add-value" bind:content={port}/>
                    </div>
                {/if}

                <div class="record-add-element">
                    <TextInput placeholder="Value" id="add-value" bind:content={value}/>
                </div>

                {#if type === "A" || type === "AAAA" }
                    <div class="record-add-element-button">
                        <ToggleButton enable={proxied} onclick={() => {proxied = !proxied}} icon="cloud"/>
                        <ToggleButton enable={showNodePinning} onclick={() => {showNodePinning = !showNodePinning}} icon="globe"/>
                    </div>
                {/if}

                <div class="record-add-element">
                    <Button inverted icon="check" onclick={() => submitForm()}>Submit</Button>
                </div>
            </div>

            <p style="margin-left: 15px; margin-bottom: 0;">
                {#if label === "@"}
                    <b>{zone}</b> points to <b>{ value ? value : "[value]" }</b>
                {:else}
                    <b>{ label ? label : "[label]" }.{zone}</b> points to <b>{ value ? value : "[value]" }</b>
                {/if}
            </p>

            {#if type === "A" || type === "AAAA" }
                <div class="info-text">
                    {#if proxied }
                        <p>This record <u>will</u> be proxied.</p>
                    {:else}
                        <p>This record <u>will not</u> be proxied.</p>
                    {/if}
                </div>

                {#if showNodePinning }
                    <div class="info-text" style="display: block">
                        <p>This record <u>will</u> be pinned to the following nodes:</p>
                        <div class="node-pinning-checkboxes" style="display: block">
                            {#if nodes}
                                <CheckBoxes nodes={nodes}/>
                            {:else}
                                <p>Loading...</p>
                            {/if}
                        </div>
                    </div>
                {/if}

            {:else}
                <div class="info-text"><p></p></div>
            {/if}
        {/if}
    </div>

    <div class="table-wrapper">
        <table class="sethjs-table">
            <tr>
                <th>Label</th>
                <th>Type</th>
                <th>TTL</th>
                <th>Value</th>
            </tr>

            {#if records}
                {#each records as record, i }
                    <tr>
                        <td>{record["label"]}</td>
                        <td>{record["type"]}</td>
                        {#if record["proxied"]}
                            <td>Auto</td>
                        {:else}
                            <td>{record["ttl"]}</td>
                        {/if}
                        <td class="flex-value">
                            {record["value"]}
                            {#if record["proxied"]}
                                <Button disabled icon="cloud">Proxied</Button>
                            {/if}
                            {#if record["pinned_nodes"]}
                                <Button disabled icon="gps_fixed">Pinned: { record["pinned_nodes"].join() }</Button>
                            {/if}
                            <Button color="red" icon="delete" size="1.25rem" onclick={() => {deleteRecord(i)}} floatRight={true}/>
                        </td>
                    </tr>
                {/each}
            {:else}
                <p style="padding-left: 10px">Loading...</p>
            {/if}
        </table>
    </div>
</main>

<style>
    main {
        margin: auto;
        border: 2px solid white;
        border-radius: 15px;
        padding-bottom: 10px;
    }

    div {
        display: flex;
        justify-content: space-between;
    }

    span {
        margin: 10px;
    }

    h2 {
        margin: 15px;
    }

    .sethjs-table {
        width: 100%;
        border-collapse: collapse;
    }

    .sethjs-table th {
        padding-top: 16px;
        padding-bottom: 16px;
        padding-left: 20px;
        text-align: left;
        background-color: #202020;
        border-bottom: 1px solid #555555;
        color: white;
        margin: 0;
    }

    .sethjs-table tr {
        width: 100%;
    }

    .sethjs-table tr:nth-child(odd) {
        background-color: #111111;
    }

    .sethjs-table td {
        padding-top: 15px;
        padding-bottom: 15px;
        padding-left: 20px;
        text-align: left;
    }

    .header-container {
        flex-direction: column;
    }

    .record-add-container {
        display: flex;
        width: calc(100% - 15px);
        align-content: center;
        flex-wrap: wrap;
        margin: 5px 8px 0;
    }

    .record-add-element {
        display: flex;
        flex-direction: column;
        margin: 5px;
        justify-content: center;
        flex: 1 1 auto;
    }

    .record-add-element-number {
        display: flex;
        flex-direction: column;
        margin: 5px;
        justify-content: center;
        flex: 0.5 0 100px;
    }

    .record-add-element-button {
        display: flex;
        margin: 5px;
    }

    .record-add-element-select {
        display: flex;
        flex-direction: column;
        margin: 5px;
        justify-content: center;
    }

    .table-wrapper {
        width: 100%;
    }

    .flex-value {
        display: flex;
        justify-content: space-between;
        align-items: center;
        word-break: break-all;
    }

    .info-text p {
        color: #afafaf;
        margin-left: 15px;
    }

    .node-pinning-checkboxes {
        margin-left: 10px;
        margin-bottom: 15px;
    }

    .capital {
        text-transform: uppercase;
    }
</style>
