FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY ["WindowsService/bin/Release/", "/Service/"]

WORKDIR "C:/Service/"

RUN New-Service -Name "SampleWindowsService" -BinaryPathName SampleWindowsService.exe; \
    Set-Service -Name "\"SampleWindowsService\"" -StartupType Automatic; \
    Set-ItemProperty "\"Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\SampleWindowsService\"" -Name AllowRemoteConnection -Value 1
    sc config SampleWindowsService obj= DOMAIN\User password= password

ENTRYPOINT ["powershell"]
CMD Start-Service \""MyWindowsServiceName\""
