
stage('Dashboard smoke test') {
    steps {
        powershell '''
            $ErrorActionPreference = "Stop"
            $html = Get-Content -Raw "index.html"

            $required = @(
                "Smart Agriculture Dashboard",
                "F-101", "F-102", "F-103", "F-104", "F-105",
                "Prioritized irrigation recommendations",
                "Water conservation plan"
            )

            foreach ($item in $required) {
                if (-not $html.Contains($item)) {
                    throw "Missing required content: $item"
                }
            }

           if (-not $html.Contains('moisture<30')) {
    throw "Missing irrigation threshold below 30%."
}

            Write-Output "Dashboard smoke test passed."
        '''
    }
}
