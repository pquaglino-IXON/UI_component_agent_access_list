<script lang="ts">
  /*
  npx cdk simulate datalogging-agents
  npx cdk deploy datalogging-agents
  npx cdk publish datalogging-agents
  */

  import type { ComponentContext } from '@ixon-cdk/types';
  import { onMount, tick } from 'svelte';
  //import { construct_svelte_component } from 'svelte/internal';
  
  export let context: ComponentContext;

  type agentUserType = { name: any; emailAddress: any; group: string; role: string; expiresOn: string};

  let rows: Array<agentUserType> = []; // rows returned from api call
  let displayRows: Array<agentUserType> = []; // actual rows displayed (filtered, sorted..)
  let statusMessage: string = ""; // status message to show to the user
  let nameSortAsc = true; //sort status, toggle
  let roleSortAsc = true; // sort status, toggle
  let groupSortAsc = false; // sort status, toggle
  let expirationSortAsc = false; // sort status, toggle
  let translations: Record<string, string>;
  let searchActive = false; // to show/hide tools when searchbox is active
  let searchInput: HTMLInputElement; // search input field, binding to manipulate it (focus, empty,..)
  let searchT: string;
  let myTranslations: Record<string, Record<string, string>>; // custom translations for the component
  let t: Record<string, string>; // actual translation based on the user's language
  let showPermanentUsers: boolean;
  let showTemporaryUsers: boolean;
  let showServiceAccounts: boolean;
  let showCompanyAccess: boolean;
  let widgetTitle: string;

  const resourceDataClient = context.createResourceDataClient();    

  onMount(async () => {
    myTranslations = (await import("./locale.json")).default;

    showPermanentUsers =  context.inputs.showPermanent;
    showTemporaryUsers =  context.inputs.showTemporary;
    showServiceAccounts = context.inputs.showServiceAccounts;
    widgetTitle =         context.inputs.title;
    showCompanyAccess =   context.inputs.showCompanyAccess;
    
    translations = context.translate(
      [
        '__TEXT__.NO_MATCHING_RESULTS',
        'SERIAL_NUMBER',
        'NAME',
        'SEARCH',
        'ROLE',

      ],
      undefined,
      { source: 'global' },
    );

    t = myTranslations[context.appData.language] || myTranslations['en'];
    searchT = t['search'];
    FetchData();
  });

  async function FetchData() {
    rows = [];
    displayRows = [];
    
    //statusMessage = t['agentlist']
    let response = await GetData();
    if (!response.status) {
      statusMessage = response.errorMsg;
      return;
    }

    rows = response.data || [];
    displayRows = rows;
    sortByName();
    statusMessage = `Found: ${rows.length}`;
  }

  async function GetData() {
    
    let resp = await queryResourceDataClient('Agent', ['name'])
    if (resp == null) {
      return {'data': null, 'errorMsg': 'Agent not found', 'status': false };
    }

    let agentId = resp['publicId']
      
    // Recupero gruppi agent
    let url = context.getApiUrl('Agent').replace('{publicId}',agentId) + '?fields=memberships.group'
    //console.log(url)
    let response = await ApiCall(url, 'GET');
    if (!response.status) 
      return {"data" : null, "status" : false, "errorMsg" : response.errorMsg };
    if (!response.data || response.data.length === 0) {
      return {"data" : null, "status" : false, "errorMsg" : 'No data returned from Agent endpoint'};
    }
    let agent = response.data[0]
    
    //console.log('Agent: \n')
    //console.log(agent)

    // Recupero lista utenti
    url = context.getApiUrl('UserList') + '?fields=name, type, emailAddress,memberships.expiresOn,memberships.group,memberships.role&page-size=4000'
    response = await ApiCall(url, 'GET');
    if (!response.status) 
      return {"data" : null, "status" : false, "errorMsg" : response.errorMsg };
    if (!response.data || response.data.length === 0) {
      return {"data" : null, "status" : false, "errorMsg" : 'No data returned from UserList endpoint'};
    }
    //let users = response.data
    const users = new Map(response.data.map((obj) => [obj['publicId'], obj]));
    // console.log('users:')
    // console.log(users)

    // Recupero lista gruppi
    url = context.getApiUrl('GroupList') + '?fields=name,agent,asset,isCompanyGroup&page-size=4000'
    response = await ApiCall(url, 'GET');
    if (!response.status) 
      return {"data" : null, "status" : false, "errorMsg" : response.errorMsg };
    if (!response.data || response.data.length === 0) {
      return {"data" : null, "status" : false, "errorMsg" : 'No data returned from GroupList endpoint'};
    }
    //let groups = response.data
    const groups = new Map(response.data.map((obj) => [obj['publicId'], obj]))
    //  console.log('groups:')
    //  console.log(groups)

    // Recupero lista ruoli
    url = context.getApiUrl('RoleList') + '?page-size=4000'
    response = await ApiCall(url, 'GET');
    if (!response.status) 
      return {"data" : null, "status" : false, "errorMsg" : response.errorMsg };
    if (!response.data || response.data.length === 0) {
      return {"data" : null, "status" : false, "errorMsg" : 'No data returned from RoleList endpoint'};
    }
    let roles = new Map(response.data.map((obj) => [obj['publicId'], obj]))
    // console.log('roles:')
    // console.log(roles)

    
    // Trova gruppi a cui l'agent appartiene
    let agentGroups = new Set<any>()
    agent['memberships'].forEach((membership: any) => {
      agentGroups.add(membership['group']['publicId'])
    })

    let agentUsers: Array<agentUserType> = [];

    // console.log('Cerco utenti agent')
    users.forEach(user => {
      let memberships = []
      //console.log('User ' + user['name'])
      if (!showServiceAccounts &&  user['type'] == 'service_account') return;
      user['memberships'].forEach((membership: any) => {
        //console.log('membership ' + membership['publicId'])
        if (membership['role'] == null) {
          //console.log('role = null')
          return;
        }
        
        let membershipGroupId = membership['group']['publicId']
        
        //console.log('check GroupId in agentGroups ' + membershipGroupId)

        if (agentGroups.has(membershipGroupId)) {
          //console.log('.       found one')
          //console.log(membership)

          let membershipGroupRole = membership['role']['publicId']
          let membershipExpiration = String(membership['expiresOn'] || '')

          if (membershipExpiration == '' && !showPermanentUsers) return;
          if (membershipExpiration != '' && !showTemporaryUsers) return;

          let groupName = (groups.has(membershipGroupId))?groups.get(membershipGroupId)['name'] || 'n/a':'-'

          if (groups.has(membershipGroupId)) {
            let g = groups.get(membershipGroupId)
            if (g['isCompanyGroup'])
              if (showCompanyAccess)
                groupName = "(Company access)"
              else
                return
            else
              groupName = g['name']
            if (groupName == null) {
              if (g['asset'] != null || g['agent'] != null) {
                groupName = '(Device specific access)'
                // console.log(g)
              }
              
            }
          }
          let roleName = (roles.has(membershipGroupRole))?roles.get(membershipGroupRole)['name'] || 'n/a':'-'

          let name = user['name'] || ''
          if (user['type'] == 'service_account') name = '[Service account] ' + name;
          
          agentUsers.push({
            'name' : name,
            'emailAddress' : user['emailAddress'] || 'n/a', 
            'group' : groupName,
            'role' : roleName,
            'expiresOn' : String(membershipExpiration)})
        }
      });
    });
    // console.log('agentUsers:')
    // console.log(agentUsers)

    return {'data': agentUsers, 'errorMsg': '', 'status': true };
  }

  async function queryResourceDataClient(selector: string | any, fields:string[]) {
    const querySelector = selector as any;
    const result = await new Promise((resolve) => {

      resourceDataClient.query([{ selector: querySelector, fields }], 
        result => { 
            resolve(result); 
        });
     
    }) as any;

    if (!result || !Array.isArray(result) || result.length === 0) {
      return null;
    }

    const data = result[0]['data'];
    if (!data || typeof data !== 'object') {
      return null;
    }

    return data;
  }

  async function ApiCall(URL: string, method: string, getFullList: boolean = true) {
    //console.log('APICall: ' + URL)
    let response = null;
    let data = null;
    let ret : Array<any> = [];
    let loop = true;
    let moreAfter : string = '';

    const headers = {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ' + context.appData.accessToken.secretId,
      'Api-Application': context.appData.apiAppId,
      'Api-Company': context.appData.company.publicId,
      'Api-Version': '2'
    }

    while (loop) {

      try {
        response = await fetch(URL + moreAfter, {headers: headers, method: method,})
      } catch (error) {
        console.error('Fetch error:\n', error);
        return {data: null, status: false, errorMsg: t['networkerror']};
      }

      try {
        data = await response.json();
      } catch (error) {
        console.error(`Error parsing JSON from\n ${URL}:\n`, error);
        return {data: null,status: false, errorMsg: t['dataerror']};
      }

      if (!data) {
        console.error(`Invalid data format:\n ${data}\n` );
        return {data: null, status: false, errorMsg: t['dataerror']};
      }
      ret = ret.concat(data.data);

      if (data.moreAfter && data.moreAfter !== '' && getFullList) {
        moreAfter = "&page-after=" + data.moreAfter
      }
      else 
        loop = false;
    }
    return {data: ret, errorMsg: '', status: true};
  }

  function navigateToAgent(agentId: string) {
    window.open('https://portal.ixon.cloud/fleet-manager/device-configurator/' + agentId, '_blank');
  }

  function refresh() {
    FetchData();
    displayRows = [...displayRows];
  }

  function exportCsv() {
    let csv = "Name,Group,Role,ExpiresOn\n";
    displayRows.forEach(user => {
      csv += `${user['name']},${user['group']},${user['role']},${user['expiresOn']}\n`;
    });

    context.saveAsFile(csv, "users.csv");
  }

  function sortByGroup() {
    displayRows.sort((a, b) => {
      return groupSortAsc ? a['group'].localeCompare(b['group']) : b['group'].localeCompare(a['group']);
    });
    groupSortAsc = !groupSortAsc;
    displayRows = [...displayRows]; // Refresh display
  }

  function sortByRole() {
    displayRows.sort((a, b) => {
      return roleSortAsc ? a['role'].localeCompare(b['role']) : b['role'].localeCompare(a['role']);
    });
    roleSortAsc = !roleSortAsc;
    displayRows = [...displayRows]; // Refresh display
  }

  function sortByName() {
    displayRows.sort((a, b) => {
      return nameSortAsc ? a['name'].localeCompare(b['name']) : b['name'].localeCompare(a['name']);
    });
    nameSortAsc = !nameSortAsc;
    displayRows = [...displayRows]; // Refresh display
  }

  function sortByExpiration() {
    displayRows.sort((a, b) => {
      return expirationSortAsc ? a['expiresOn'].localeCompare(b['expiresOn']) : b['expiresOn'].localeCompare(a['expiresOn']);
    });
    expirationSortAsc = !expirationSortAsc;
    displayRows = [...displayRows]; // Refresh display
  }

  async function showSearchbox(open: boolean) {
    searchActive = open;
    if (open) {
      await tick(); // Wait for the DOM to update
      searchInput.focus();
    } else {
      searchInput.value = '';
      search('');
    }
  }

  function handleSearchInput(e: Event) {
    const target = e.target as HTMLInputElement;
    if (target) {
      search(target.value);
      statusMessage = `${t['found']}: ${rows.length}`;
    }
  }

  async function search(searchTerm: string) {
    if (searchTerm.trim() === '') {
      displayRows = rows;
      await tick();
      statusMessage = `${t['found']}: ${displayRows.length}`;
    } else {
      const lowerSearchTerm = searchTerm.toLowerCase();
      displayRows = rows.filter(agent => 
        agent['name'].toLowerCase().includes(lowerSearchTerm) || 
        agent['group'].toLowerCase().includes(lowerSearchTerm) ||
        agent['role'].toLowerCase().includes(lowerSearchTerm)
      );
      await tick();
      statusMessage = `${t['found']}: ${displayRows.length}`;
    }
  }
</script>
<main>
  <div class="card">
    <div class="card-header with-actions">
      <h3 class="card-title">{widgetTitle}</h3>
      <div class="actions-top">
        <div class="{searchActive ? '' : 'hidden'}">
          <div class="search-box">
            <input type="text" style="outline:none" placeholder="{searchT}" class="search-input" bind:this={searchInput} on:input={handleSearchInput}/>
            <div class="search-input-suffix">
              <button on:click={() => showSearchbox(false)} class="icon-button">
                <svg width="20" height="20" viewBox="0 0 24 24"><path d="M0 0h24v24H0z" fill="none"></path><path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"></path></svg>
              </button>
            </div>
          </div>
        </div>
        <button on:click={() => showSearchbox(true)} class="icon-button toolbox {searchActive ? 'hidden' : ''}"><svg width="24" height="24" viewBox="0 0 24 24"><path d="M0 0h24v24H0z" fill="none"></path><path d="M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3 9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z"></path></svg></button>
        <button on:click={() => exportCsv()} class="icon-button toolbox {searchActive ? 'hidden' : ''}"><svg width="24" height="24" viewBox="0 0 24 24"><path d="M4 15H6V18H18V15H20V18C20 19.1 19.1 20 18 20H6C4.9 20 4 19.1 4 18V15Z"></path><path d="M13 12.17L15.59 9.59L17 11L12 16L7 11L8.41 9.58L11 12.17L11 4L13 4L13 12.17Z"></path></svg></button>
        <button on:click={() => refresh()} class="icon-button toolbox {searchActive ? 'hidden' : ''}"><svg width="24" height="24" viewBox="0 0 28 25"><path d="M 12 2 C 6.486 2 2 6.486 2 12 C 2 17.514 6.486 22 12 22 C 17.514 22 22 17.514 22 12 L 20 12 C 20 16.411 16.411 20 12 20 C 7.589 20 4 16.411 4 12 C 4 7.589 7.589 4 12 4 C 14.205991 4 16.202724 4.9004767 17.650391 6.3496094 L 15 9 L 22 9 L 22 2 L 19.060547 4.9394531 C 17.251786 3.1262684 14.757292 2 12 2 z"></path></svg></button>
      </div>
    </div>
    <div class="card-content has-header">
      {#if displayRows && displayRows.length > 0}
      <div class="table-container">
        <table class="table datalogging-agents-table">
          <thead>
            <tr>
              <th on:click={() => sortByName()}>Name</th>
              <th on:click={() => sortByGroup()}>Group</th>
              <th on:click={() => sortByRole()}>Role</th>
              <th on:click={() => sortByExpiration()}>Expiration</th>
            </tr>
          </thead>
          <tbody>
            {#each displayRows as agentUser}
                <tr>
                  <td>{agentUser.name}</td>
                  <td>{agentUser.group}</td>
                  <td>{agentUser.role}</td>
                  <td>{agentUser.expiresOn}</td>
                </tr>
            {/each}
          </tbody>
        </table>
      </div>
      {/if}
      <p>{statusMessage}</p>
    </div>
  </div>
</main>

<style lang="scss">
  @import 'card';

  .card {
    color: var(--card-color);

    .card-header {
      svg {
        fill: currentColor;
        stroke: none;
      }

      .hidden {
        display: none;
      }

      button.icon-button {
        background-color: transparent;
        border: none;
        cursor: pointer;
      }

      div.search-box {
        background-color: color-mix(in srgb, transparent, currentcolor 4%);
        border-radius: 20px;
        padding-left: 15px;
        display: inline-block;
        vertical-align: top;

        input.search-input {
          outline: none;
          background-color: transparent;
          height: 20px;
          width: 100px;
          padding: 4px 8px 4px 0;
          margin: 0;
          border: none;
          line-height: 24px;
          font-size: 14px;
          color: currentcolor;
        }

        div.search-input-suffix {
          display: inline-block;

          button {
            position: relative;
            top: 4px;
          }
        }
      }    
    }

    .card-content {

      h3 {
        margin-bottom: 1em;
      }

      p {
        margin: 0;
      }

      .table-container {
        overflow: auto;
        overflow-anchor: none;
        position: absolute;
        left:8px;
        right:8px;
        top:65px;
        bottom:0;
      
        table.datalogging-agents-table {
          width: 100%;
          border-collapse: collapse;

          a {
            text-decoration: none;
            color: inherit;
            cursor: pointer;
          }

          th {
            position: sticky;
            z-index: +10;
            top: 0;
            background-color: var(--card-bg);
          }
          
          tr {
            text-align: left;
            line-height: 24px;
            //cursor: pointer;

            td {
              padding-right: 2em;
            }
          }

          thead tr th{
            cursor: pointer;
          }

          tbody tr {
            border-bottom: 1px solid color-mix(in srgb, transparent, currentcolor 12%);;
            border-collapse: collapse;
            
            &:hover {
              background-color: var(--accent);
              color: var(--accent-color);
            }
          }
        }
      }
    }
  }
</style>
