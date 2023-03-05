# 如何使用 Octopus Deploy 从《我的世界》部署- Octopus Deploy

> 原文：<https://octopus.com/blog/how-to-deploy-from-minecraft-with-octopus-deploy>

想象一个部署简单的世界。你可以在你最喜欢的视频游戏中按下一个按钮，你的最新版本就可以投入生产了。有些人可能会嘲笑:“我的电子表格，RDP 和手动配置文件编辑永远不会被一个虚拟按钮取代！”请允许我介绍 *OctoCraft 展开*:

[https://www.youtube.com/embed/RjUIQmdIlEc](https://www.youtube.com/embed/RjUIQmdIlEc)

VIDEO

在《我的世界》这边，你需要的只是一个 Bukkit 和一份 T2《我的世界》的拷贝。Bukkit 允许创建自定义的《我的世界》插件。 *OctoCraft Deploy* 只是一个与 Octopus Deploy API 交互的《我的世界》插件。

Octopus Deploy 是 [API first](http://docs.octopusdeploy.com/display/OD/Octopus+REST+API) 。您可以通过 UI 做的任何事情都可以通过 API 来完成。即使用 Java。 *OctoCraft Deploy* 调用 Octopus Deploy API 来创建一个发布，将这个发布部署到一个环境中，然后监控部署的状态。你可以在这里找到完整的 [API 文档](https://github.com/OctopusDeploy/OctopusDeploy-Api/wiki)。

例如，要创建发行版:

创建一个发布到 API 的方法。

```
public HttpResponse Post(String path, String json) throws ClientProtocolException, IOException {
    HttpClient client = new DefaultHttpClient();
    HttpPost post = new HttpPost(url + path);
    post.setHeader(API_KEY_HEADER, apiKey);

    post.setEntity(new StringEntity(json, ContentType.TEXT_PLAIN));
    return client.execute(post);
} 
```

将序列化到 post 请求的 POJO。

```
public class ReleasePost {
    private String projectId;
    private String version;

    public ReleasePost(String projectId, String version) {
        this.projectId = projectId;
        this.version = version;
    }

    @JsonProperty("ProjectId")
    public String getProjectId() {
        return projectId;
    }

    @JsonProperty("Version")
    public String getVersion() {
        return version;
    }
} 
```

做好创建发布的所有艰苦工作。

```
private Release createRelease(Project project) throws ClientProtocolException, IOException {                
    ReleasePost releasePost = new ReleasePost(project.getId(), "1.i");
    String content = new String(); 
    String json = objectMapper.writeValueAsString(releasePost);
    HttpResponse response = webClient.Post(RELEASES_PATH, json);

    content = EntityUtils.toString(response.getEntity());           
    return objectMapper.readValue(content, Release.class);  
} 
```

GitHub 上的[完整源代码。](https://github.com/tothegills/OctocraftDeploy)