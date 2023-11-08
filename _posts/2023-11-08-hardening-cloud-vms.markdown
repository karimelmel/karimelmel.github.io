---
layout: post
title:  "Hardening Cloud VMs"
date:   2023-11-08
---

My career in cybersecurity started with OS hardening, so I am rather passionate about the topic. When it comes to the cloud, many organizations have embraced automation. Yet, most skip hardening of the operating system or only do a bare minimum hardening baseline. I've also seen the pre-hardened CIS images used, but to my frustration, it update less frequently than other images. I am writing this as a companion post to my talk at NICConf. 

So, what are the options for hardening images in the cloud?

**The Microsoft way**

Microsoft provides the Security Compliance Toolkit (SCT), a collection of tools and Group Policy Object Backups that allow the hardening of Windows images. The solution is easy to use but has two large drawbacks

1) The package includes an executable (LGPO.exe) to apply the Group Policy Backup Objects

2) The Group Policy Backup Objects are a pain to modify. Configuring the policy not to run the baseline as-is is a hassle, and its not worth adapting it to a diverse set of workloads with different incompatibility issues.

**HardeningKitty**

While considering building my solution based on the different customizations to 'The Microsoft way' I have done over the years, I came across a project that had already done that to the full extent. Rather than Group Policy Backups, HardeningKitty provides a PowerShell module and leverages a CSV file where you can modify the settings and audit the state. This approach gives hassle-free modifications, lightweight packaging, and visibility https://github.com/scipag/HardeningKitty

It also comes with a wide variety of pre-defined baselines.

**Running HardeningKitty on your cloud workloads**

There is no good reason to go for the Microsoft Way here, and for the rest of this post, I assume you will choose to run with HardeningKitty.

There are plenty of ways to execute code on your cloud workloads, and what approach is the best for you depends on multiple factors:

- Do you bake golden images?
- Do you provision everything upon deployment?
- Do you use userdata or Custom Script Extensions?
- Do you have a way of managing packages and artifacts?

Let's say you don't care. You want to test the hardening.

Depending on the answers, you would know best where and how to run it. I've run it with artifacts in various solutions, such as simple blob storage to solutions, such as Artifactory, and in multiple processes, such as image pipelines and userdata. Regardless of how you run it, it does the job.

The script below is just a simple wrapper that gets the HardeningKitty from its source and runs it with one of the predefined configuration files.

```ps1
<#
.SYNOPSIS
    Function for downloading and executing HardeningKitty
    https://github.com/scipag/HardeningKitty

.DESCRIPTION
    Triggers three distinctive functions as a single line to apply hardening and passing the parameters.
    
.PARAMETER FileFindingList
    The path to the CSV file for HardeningKitty configuration.

.PARAMETER HardeningKittyPath
    The path to where HardeningKitty module is imported from.

.PARAMETER UnzipPath
    The path to where the downloaded file is unzipped to.

.PARAMETER PackageUrl
    The URL to the zip package to download and extract.

.NOTES
    All parameters are passed in the super-function to the corresponding functions. 

.EXAMPLE 
    Invoke-Hardening
    Invoke-Hardening -FileFindingList <path to custom file finding list>
#>

function Invoke-Hardening {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false)]
        [string]
        $FileFindingList = (Join-Path -Path $env:TEMP -ChildPath "SecurityBaseline\HardeningKitty-v.0.9.0\lists\finding_list_0x6d69636b_machine.csv"),
        [string]
        $HardeningKittyPath = ( Join-Path $env:TEMP -ChildPath "SecurityBaseline\HardeningKitty-v.0.9.0" ),
        [Parameter(Mandatory = $false)]
        [string]
        $UnzipPath = ( Join-Path $env:TEMP -ChildPath "SecurityBaseline" ),
        [Parameter(Mandatory = $false)]
        [string]
        $PackageUrl = "https://github.com/scipag/HardeningKitty/archive/refs/tags/v.0.9.0.zip"
    )

    function Get-UnzippedPackage {
        param(
            [Parameter(Mandatory = $true)]
            [string]
            $PackageUrl,
            [Parameter(Mandatory = $true)]
            [string]
            $UnzipPath
        )
        try {

            Write-Information -MessageData "Downloading the zip package from the $PackageUrl"
            $package = Invoke-WebRequest $PackageUrl -UseBasicParsing

            Write-Information -MessageData "Creating a new temporary directory"
            $tempDir = New-Item -ItemType Directory -Path (Join-Path $env:TEMP ([System.Guid]::NewGuid().ToString()))

            Write-Information -MessageData "Saving the package content to a temporary file"
            $tempFile = Join-Path $tempDir.FullName "package.zip"
            [IO.File]::WriteAllBytes($tempFile, $package.Content)
        
            Write-Information -MessageData "Extracting the contents of the zip file to the destination directory"
            Expand-Archive -Path $tempFile -DestinationPath $UnzipPath -Force

            Write-Information -MessageData "Removing the temporary directory and its contents"
            Remove-Item $tempDir.FullName -Recurse -Force
        }
        catch {
            Write-Error -Message "Failed to download and unzip package from $Url. $_"
        }
    }

    function Invoke-HardeningKittyHelper {
        [CmdletBinding()]
        param (
            [Parameter(Mandatory = $true)]
            [string]
            $FileFindingList,
            [Parameter(Mandatory = $true)]
            [string]
            $HardeningKittyPath
        )
        try {
            Write-Information -MessageData "Importing the HardeningKitty module"
            Import-Module -Name (Join-Path -Path $HardeningKittyPath -ChildPath "HardeningKitty.psm1") -ErrorAction Stop
        }
        catch {
            Write-Error -Message "Failed to import module from $HardeningKittyPath. $_"
            return
        }
    
        try {
            Write-Information -MessageData "Invoking the HardeningKitty script with the FileFindingList provided"
            Invoke-HardeningKitty -FileFindingList $FileFindingList -Mode HailMary -Log -Report -SkipRestorePoint
        }
        catch {
            Write-Error -Message "Failed to run Invoke-HardeningKitty. $_"
        }
    }

    $GetUnzippedPackageParams = @{ 
        PackageUrl = $PackageUrl 
        UnzipPath  = $UnzipPath
    }
    Get-UnzippedPackage @GetUnzippedPackageParams

    $InvokeHardeningKittyHelperParams = @{
        FileFindingList    = $FileFindingList 
        HardeningKittyPath = $HardeningKittyPath
    }
    Invoke-HardeningKittyHelper @InvokeHardeningKittyHelperParams
}

Invoke-Hardening
```

With this approach, it is portable to the point where I can run complete hardening through Azure Run Command, which you should never do in a real environment. It just demonstrates the portability.

![image](https://github.com/karimelmel/karimelmel.github.io/assets/26272119/1be5f73f-b90f-403f-bda5-04c37be5109d)

That's all for this post.