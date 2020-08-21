---
layout: post
title: ________
date: YYYY-MM-DD
categories: [placeholder]
tags: [placeholder]
---

Intro paragraph (this will be used as the synopsis/summary/excerpt on the post listing pages)

## First Heading

Some text.

```powershell
Set-AWSCredential -AccessKey $AccessKey -SecretKey $SecretKey -Scope Local

$InstanceId = $null
# Windows 2019
$AmiId = 'ami-0f38562b9d4de0dfe'

$keyPairGuid = (New-Guid).Guid
$keyPairFile = Join-Path ([System.IO.Path]::GetTempPath()) -ChildPath "keypair-$keyPairGuid.pem"

$keyPair = New-EC2KeyPair -KeyName $keyPairGuid
$keyPair.KeyMaterial | Set-Content -Path $keyPairFile

try {
    $Region = 'us-east-1'
    $SubnetList = @(
        @{ AvailabilityZone = 'us-east-1c'; SubnetId = $subnetA }
        @{ AvailabilityZone = 'us-east-1a'; SubnetId = $subnetB }
    )

    $InstanceTypeList = @(
        @{ Name = 'z1d.metal'; Price = 4 }
        @{ Name = 'i3.metal'; Price = 4.5 }
        @{ Name = 'c5d.metal'; Price = 6 }
        @{ Name = 'm5d.metal'; Price = 6.1 }
        @{ Name = 'r5d.metal'; Price = 6.2 }
        @{ Name = 'g4dn.metal'; Price = 6.77 }
    )

    $instanceTags = [Amazon.EC2.Model.TagSpecification]@{
        Tags         = [Amazon.EC2.Model.Tag[]]@(
            @{ Key = 'Name'; Value = "My Spot Instance" }
            @{ Key = 'created-by'; Value = 'creating-user' }
        )
        ResourceType = [Amazon.EC2.ResourceType]::SpotInstancesRequest
    }

    # Try available regions, subnets, and preferred instance types in order
    # until an appropriate instance is made available.
    $InstanceId = :main foreach ($subnet in $SubnetList) {
        # Spot requests don't usually take a long time to fulfil.
        # We're better off just trying another instance type / zone and taking
        # what's first available in our price preference ranges.
        $ValidUntil = [DateTime]::UTcNow.AddMinutes(2)
        $spotRequest = $null

        foreach ($InstanceType in $InstanceTypeList) {
            $spotRequestParams = @{
                InstanceCount                                  = 1
                Region                                         = $Region
                SpotPrice                                      = $InstanceType.Price
                TagSpecification                               = $instanceTags
                Type                                           = 'one-time'
                UtcValidUntil                                  = $ValidUntil
                LaunchSpecification_ImageId                    = $AmiId
                LaunchSpecification_InstanceType               = $InstanceType.Name
                LaunchSpecification_KeyName                    = $key.KeyName
                LaunchSpecification_Placement_AvailabilityZone = $subnet.AvailabilityZone
                LaunchSpecification_SecurityGroup              = $SecurityGroup
                LaunchSpecification_SubnetId                   = $subnet.SubnetId
                IamInstanceProfile_Name                        = $IamName
            }
            $spotRequest = Request-EC2SpotInstance @spotRequestParams

            try {
                do {
                    Start-Sleep -Seconds 5
                    $spotRequest = Get-EC2SpotInstanceRequest -SpotInstanceRequestId $spotRequest.SpotInstanceRequestId
                    $spotRequest.Status | Out-Host
                } while ($spotRequest.State -eq [Amazon.EC2.SpotInstanceState]::Open -and $spotRequest.Status.Code -ne 'capacity-not-available')

                if ($spotRequest.InstanceId) {
                    $spotRequest.InstanceId
                    break main
                }
            }
            finally {
                Write-Host $spotRequest.Status.Message
                $null = $spotRequest | Stop-EC2SpotInstanceRequest
            }
        }
    }

    if (-not $InstanceId) {
        throw "Could not get a spot instance allocated"
    }

    # Ensure we tag the instance and any storage volumes correctly
    $Instance = Get-EC2Instance -InstanceId $InstanceId -Select 'Reservations.Instances'
    $blockDeviceIds = $Instance.BlockDeviceMappings.Ebs.VolumeId
    New-EC2Tag -Resource $InstanceId -Tag $instanceTags.Tags
    if ($blockDeviceIds) {
        New-EC2Tag -Resource $blockDeviceIds -Tag $instanceTags.Tags
    }

    $timer = [System.Diagnostics.Stopwatch]::StartNew()
    $password = $null

    do {
        Write-Host "Waiting for password to be generated... $($timer.Elapsed)"
        Start-Sleep -Seconds 30
        $password = Get-EC2PasswordData -InstanceId $instanceId -Decrypt -PemFile $keyPairFile | ConvertTo-SecureString -AsPlainText -Force
    } while (-not $password)

    $timer.Stop()
    Write-Host "Password generated in $($timer.Elapsed)"

    $credential = [pscredential]::new('Administrator', $password)

    $password = $null

    # Enable PSRemoting via Systems Manager document command
    $ssmCommandParams = @{
        InstanceId                                     = $InstanceId
        Comment                                        = "Enable PSRemoting for instance $InstanceId"
        DocumentName                                   = $SsmDocumentName
        Parameter                                      = @{
            IPAddress = (Invoke-WebRequest -Uri 'http://ipinfo.io/ip').Content
        }
        Region                                         = 'us-east-1'
        DocumentVersion                                = '$LATEST'
        CloudWatchOutputConfig_CloudWatchOutputEnabled = $true
        CloudWatchOutputConfig_CloudWatchLogGroupName  = 'my-log-group-name'
    }
    $commandDetails = Send-SSMCommand @ssmCommandParams

    Write-Host "Waiting for $SsmDocumentName command document to be processed."
    do {
        Start-Sleep -Seconds 5
        $commandDetails = Get-SSMCommandInvocation -CommandId $commandDetails.CommandId
    } while ($commandDetails.StatusDetails -in 'Pending', 'InProgress')

    if ($commandDetails.StatusDetails -ne 'Success') {
        throw "Unable to enable PSRemoting on instance <$InstanceId>. Check CloudWatch logs in log group $($ssmCommandParams.CloudWatchOutputConfig_CloudWatchLogGroupName) for details"
    }

    $sessionOptions = New-PSSessionOption -SkipCACheck -SkipCNCheck
    $session = New-PSSession -Credential $credential -ComputerName $Instance.PublicDnsName -SessionOption $sessionOptions -UseSSL
}
finally {
    # Cleanup - terminate the instances
    Remove-EC2KeyPair -KeyName $keyPair.KeyName -Force
    Remove-EC2Instance -InstanceId $instanceId -Force |
        Select-Object -Property @{ Name = 'InstanceId'; Expression = { $InstanceId } } -ExpandProperty 'CurrentState' |
        Out-Host

    Remove-Item -Path $keyPairFile -Force
}

```

```yaml
---
schemaVersion: "2.2"
description: Enables PSRemoting and adds the requested IP address to TrustedHosts in order to facilitate PSRemoting connections from that machine.
parameters:
  IPAddress:
    type: String
    description: The IP Address to add to TrustedHosts.
    default: "*"

mainSteps:
  - action: "aws:runPowerShellScript"
    name: AddIPAddressToTrustedHosts
    inputs:
      runCommand:
      - $trustedHosts = (Get-Item -Path 'WSMan:\localhost\Client\TrustedHosts').Value
      - $trustedHosts = @($trustedHosts, '{{IPAddress}}' | Select-Object -Unique | Where-Object {$_}) -join ', '
      - Set-Item -Path 'WSMan:\localhost\Client\TrustedHosts' -Value $trustedHosts -Force -ErrorAction Stop
      - Enable-PSRemoting -Force -SkipNetworkProfileCheck
      - $cert = New-SelfSignedCertificate  -DnsName "$env:COMPUTERNAME" -KeyAlgorithm RSA -KeyLength 2048 -CertStoreLocation "cert:\LocalMachine\My"
      - New-NetFirewallRule -Direction Inbound -LocalPort 5986 -Protocol TCP -Action Allow -DisplayName "WinRM-HTTPS"
      - New-WSManInstance winrm/config/Listener -SelectorSet @{Transport='HTTPS'; Address='*'} -ValueSet @{Hostname=$env:COMPUTERNAME;CertificateThumbprint=$cert.Thumbprint}
```
