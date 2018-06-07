FROM gcr.io/google-appengine/aspnetcore:2.0
ADD /var/jenkins_home/workspace/re-app-to-Kubernetes_master-TOAR4D3XJM34VMJ5QV2UK762X57QHHUJUDWEJBOBVA6I6YUW3RCQ/bin/Release/netcoreapp2.0/publish /app
ENV ASPNETCORE_URLS=http://*:${PORT}
WORKDIR /app
ENTRYPOINT [ "dotnet", "HelloWorldAspNetCore.dll"]
