name: Multi-Repository Clone and Execute

on:
  workflow_dispatch:
    inputs:
      repo1_url:
        description: 'First repository URL (e.g., https://github.com/owner/repo1.git)'
        required: true
        type: string
      repo1_ref:
        description: 'Branch/tag/commit for first repo'
        required: false
        default: 'main'
        type: string
      repo2_url:
        description: 'Second repository URL (e.g., https://github.com/owner/repo2.git)'
        required: true
        type: string
      repo2_ref:
        description: 'Branch/tag/commit for second repo'
        required: false
        default: 'main'
        type: string
      commands:
        description: 'Commands to execute after cloning (one per line)'
        required: true
        type: string
jobs:
  multi-repo-clone:
    name: Clone repositories and execute commands
    runs-on: windows-latest
    
    env:
      REPO1_PATH: ${{ github.workspace }}\repo1
      REPO2_PATH: ${{ github.workspace }}\repo2  
  
    steps:
      - name: Clone first repository
        uses: actions/checkout@v4
        with:
          repository: ${{ github.event.inputs.repo1_url }}
          ref: ${{ github.event.inputs.repo1_ref }}
          path: repo1
          token: ${{ secrets.CUSTOM_TOKEN || secrets.GITHUB_TOKEN }}      

      - name: Clone second repository
        uses: actions/checkout@v4
        with:
          repository: ${{ github.event.inputs.repo2_url }}
          ref: ${{ github.event.inputs.repo2_ref }}
          path: repo2
          token: ${{ secrets.CUSTOM_TOKEN || secrets.GITHUB_TOKEN }}
      
      - name: Verify repository clones
        shell: powershell
        run: |
          Write-Host "Verifying repository clones..."
          
          if (-not (Test-Path "repo1")) {
            Write-Error "Failed to clone first repository: ${{ github.event.inputs.repo1_url }}"
            exit 1
          }
          
          if (-not (Test-Path "repo2")) {
            Write-Error "Failed to clone second repository: ${{ github.event.inputs.repo2_url }}"
            exit 1
          }
          
          Write-Host "✓ First repository cloned successfully at: $(Resolve-Path 'repo1')"
          Write-Host "✓ Second repository cloned successfully at: $(Resolve-Path 'repo2')"
          
          # Display basic info about cloned repositories
          Write-Host ""
          Write-Host "Repository 1 contents:"
          Get-ChildItem "repo1" | Select-Object Name, Mode | Format-Table -AutoSize
          
          Write-Host "Repository 2 contents:"
          Get-ChildItem "repo2" | Select-Object Name, Mode | Format-Table -AutoSize
      
      - name: Execute custom commands
        shell: powershell
        run: |
          # Set error action preference to stop on any error
          $ErrorActionPreference = 'Stop'
          
          # Display repository paths for debugging
          Write-Host "Repository 1 path: $env:REPO1_PATH"
          Write-Host "Repository 2 path: $env:REPO2_PATH"
          
          # Verify repositories were cloned successfully
          if (-not (Test-Path $env:REPO1_PATH)) {
            throw "Repository 1 was not cloned successfully at path: $env:REPO1_PATH"
          }
          
          if (-not (Test-Path $env:REPO2_PATH)) {
            throw "Repository 2 was not cloned successfully at path: $env:REPO2_PATH"
          }
          
          Write-Host "Both repositories cloned successfully. Executing custom commands..."     
     
          # Parse and execute commands
          $commands = @"
          ${{ github.event.inputs.commands }}
          "@
          
          $commandLines = $commands -split "`n" | Where-Object { $_.Trim() -ne "" }
          
          if ($commandLines.Count -eq 0) {
            Write-Host "No commands provided to execute."
            exit 0
          }
          
          Write-Host "Found $($commandLines.Count) command(s) to execute:"
          for ($i = 0; $i -lt $commandLines.Count; $i++) {
            Write-Host "  $($i + 1). $($commandLines[$i].Trim())"
          }
          
          # Execute each command
          for ($i = 0; $i -lt $commandLines.Count; $i++) {
            $command = $commandLines[$i].Trim()
            if ($command -ne "") {
              Write-Host ""
              Write-Host "Executing command $($i + 1): $command"
              Write-Host "----------------------------------------"
              
              try {
                Invoke-Expression $command
                Write-Host "Command $($i + 1) completed successfully."
              }
              catch {
                Write-Error "Command $($i + 1) failed: $command"
                Write-Error "Error: $($_.Exception.Message)"
                throw
              }
            }
          }
          
          Write-Host ""
          Write-Host "All commands executed successfully!"
