# Retrieve the current user-level PATH variable
$currentPath = [Environment]::GetEnvironmentVariable('PATH', [EnvironmentVariableTarget]::User)

# Convert it into an array splitting at each semicolon
$pathArray = $currentPath -split ';'

# The path you want to remove
$pathToRemove = "C:\Path\To\Remove"

# Remove the unwanted path
$newPathArray = $pathArray | Where-Object { $_ -ne $pathToRemove }

# Combine the remaining paths back into a single string separated by semicolons
$newPath = $newPathArray -join ';'

# Set the new user-level PATH variable
[Environment]::SetEnvironmentVariable('PATH', $newPath, [EnvironmentVariableTarget]::User)


