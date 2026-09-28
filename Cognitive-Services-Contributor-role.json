## BEGIN: VARIABLES SECTION INSERT START
$objectId          = $Request.Body.objectId
$tenant            = $Request.Body.tenant
$subscriptionId    = $Request.Body.subscriptionId
$resourceGroupName = $Request.Body.resourceGroupName   # optional - omit to assign at subscription scope
$objectType        = $Request.Body.objectType          # optional - 'User' (default), 'ServicePrincipal', or 'Group'
## END: VARIABLES SECTION INSERT END

Import-Module -Name Az.Accounts
Import-Module -Name Az.Resources

$tenant = $sysAddedTenantId

if (-not $objectId) {
    Write-Host "ERROR: objectId is missing from the request body."
    return
}

if (-not $subscriptionId) {
    Write-Host "ERROR: subscriptionId is missing from the request body."
    return
}

if (-not $objectType) { $objectType = 'User' }

$roleName = 'Cognitive Services Contributor'

# Build scope
if ($resourceGroupName) {
    $scope = "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName"
} else {
    $scope = "/subscriptions/$subscriptionId"
}

Write-Host "Tenant: $tenant"
Write-Host "Subscription: $subscriptionId"
Write-Host "Scope: $scope"
Write-Host "Assigning '$roleName' to objectId: $objectId (Type: $objectType)"

try {
    # Connect to Azure using current client ID and secret
    Connect-AzAccount -ServicePrincipal -Tenant $tenant -Credential $Credential -Subscription $subscriptionId -ErrorAction Stop | Out-Null

    # Validate role definition exists
    $roleDef = Get-AzRoleDefinition -Name $roleName -ErrorAction SilentlyContinue
    if (-not $roleDef) {
        Write-Host "ERROR: Role definition '$roleName' not found in subscription $subscriptionId."
        return
    }

    # Check for an existing assignment to keep the script idempotent
    $existing = Get-AzRoleAssignment -ObjectId $objectId -RoleDefinitionName $roleName -Scope $scope -ErrorAction SilentlyContinue |
                Where-Object { $_.Scope -eq $scope }

    if ($existing) {
        Write-Host "INFO: '$roleName' is already assigned to $objectId at $scope (AssignmentId: $($existing.RoleAssignmentId)). Skipping."
        return
    }

    try {
        $assignment = New-AzRoleAssignment `
            -ObjectId $objectId `
            -ObjectType $objectType `
            -RoleDefinitionName $roleName `
            -Scope $scope `
            -ErrorAction Stop

        Write-Host "SUCCESS: Assigned '$roleName' (AssignmentId: $($assignment.RoleAssignmentId)) to $objectId at $scope"
    }
    catch {
        if ($_.Exception.Message -match 'Conflict|RoleAssignmentExists|already exists') {
            Write-Host "INFO: '$roleName' already exists for $objectId at $scope."
        } else {
            Write-Host "ERROR assigning '$roleName': $($_.Exception.Message)"
        }
    }
}
catch {
    Write-Host "ERROR: $($_.Exception.Message)"
    throw
}
finally {
    # Disconnect from Azure
    Disconnect-AzAccount | Out-Null
}
