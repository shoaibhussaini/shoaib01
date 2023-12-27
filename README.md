$path = [Environment]::GetEnvironmentVariable('PATH', [EnvironmentVariableTarget]::Machine)
$gitPath = 'C:\Program Files\Git\bin'  # Update this to the path where Git 2.43 is located
$newPath = $gitPath + ';' + $path.Replace($gitPath + ';', '')
[Environment]::SetEnvironmentVariable('PATH', $newPath, [EnvironmentVariableTarget]::Machine)
