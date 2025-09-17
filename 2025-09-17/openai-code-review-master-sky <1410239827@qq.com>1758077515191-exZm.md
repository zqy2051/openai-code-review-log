# 小傅哥项目： OpenAi 代码评审.
### 😀代码评分：85
#### 😀代码逻辑与目的：
该代码实现了一个基于OpenAI（ChatGLM）的代码评审工具，通过GitHub Actions触发。主要功能包括：获取Git差异代码、调用ChatGLM API进行代码评审、将评审结果推送到GitHub仓库并通过微信发送通知。重构后采用了DDD分层架构，提高了代码的可维护性和扩展性。
#### ✅代码优点：
1. 采用DDD分层架构，职责分离清晰
2. 引入接口和抽象类，符合依赖倒置原则
3. 使用try-with-resources管理资源，防止内存泄漏
4. 添加了SLF4J日志框架，便于问题追踪
5. 通过环境变量管理敏感配置，提高安全性
#### 🤔问题点：
1. **硬编码配置**：OpenAiCodeReview类中存在硬编码的微信和ChatGLM配置
2. **异常处理不完善**：AbstractOpenAiCodeReviewService的exec方法捕获所有异常后仅记录日志，未向上抛出
3. **性能问题**：OpenAiCodeReviewService中提示词字符串过长，每次请求都发送大量数据
4. **边界条件处理不足**：GitCommand的diff方法未处理无提交记录的情况
5. **资源管理**：GitCommand的commitAndPush方法未使用try-with-resources管理Git对象
6. **命名规范**：OpenAiCodeReview类中成员变量命名不符合驼峰命名法
7. **代码重复**：TemplateMessageDTO中put方法实现重复
#### 🎯修改建议：
1. 移除硬编码配置，全部从环境变量读取
2. 在AbstractOpenAiCodeReviewService的exec方法中添加异常重新抛出机制
3. 将长提示词提取为配置文件或常量类
4. 在GitCommand的diff方法中添加提交记录存在性检查
5. 使用try-with-resources管理Git对象
6. 修正OpenAiCodeReview类中的成员变量命名
7. 抽取TemplateMessageDTO中重复的put方法实现
#### 💻修改后的代码：
```java
// OpenAiCodeReview.java
public class OpenAiCodeReview {
    private static final Logger logger = LoggerFactory.getLogger(OpenAiCodeReview.class);

    // 微信配置
    private final String weixinAppid = getEnv("WEIXIN_APPID");
    private final String weixinSecret = getEnv("WEIXIN_SECRET");
    private final String weixinTouser = getEnv("WEIXIN_TOUSER");
    private final String weixinTemplateId = getEnv("WEIXIN_TEMPLATE_ID");

    // ChatGLM 配置
    private final String chatGlmApiHost = getEnv("CHATGLM_APIHOST");
    private final String chatGlmApiKeySecret = getEnv("CHATGLM_APIKEYSECRET");

    // Github 配置
    private final String githubReviewLogUri = getEnv("GITHUB_REVIEW_LOG_URI");
    private final String githubToken = getEnv("GITHUB_TOKEN");

    // 工程配置 - 自动获取
    private final String githubProject = getEnv("COMMIT_PROJECT");
    private final String githubBranch = getEnv("COMMIT_BRANCH");
    private final String githubAuthor = getEnv("COMMIT_AUTHOR");
    private final String githubMessage = getEnv("COMMIT_MESSAGE");

    public static void main(String[] args) throws Exception {
        GitCommand gitCommand = new GitCommand(
                getEnv("GITHUB_REVIEW_LOG_URI"),
                getEnv("GITHUB_TOKEN"),
                getEnv("COMMIT_PROJECT"),
                getEnv("COMMIT_BRANCH"),
                getEnv("COMMIT_AUTHOR"),
                getEnv("COMMIT_MESSAGE")
        );

        WeiXin weiXin = new WeiXin(
                getEnv("WEIXIN_APPID"),
                getEnv("WEIXIN_SECRET"),
                getEnv("WEIXIN_TOUSER"),
                getEnv("WEIXIN_TEMPLATE_ID")
        );

        IOpenAI openAI = new ChatGLM(getEnv("CHATGLM_APIHOST"), getEnv("CHATGLM_APIKEYSECRET"));

        OpenAiCodeReviewService openAiCodeReviewService = new OpenAiCodeReviewService(gitCommand, openAI, weiXin);
        openAiCodeReviewService.exec();

        logger.info("openai-code-review done!");
    }

    private static String getEnv(String key) {
        String value = System.getenv(key);
        if (null == value || value.isEmpty()) {
            throw new RuntimeException("Environment variable " + key + " is not set");
        }
        return value;
    }
}

// AbstractOpenAiCodeReviewService.java
@Override
public void exec() {
    try {
        // 1. 获取提交代码
        String diffCode = getDiffCode();
        // 2. 开始评审代码
        String recommend = codeReview(diffCode);
        // 3. 记录评审结果；返回日志地址
        String logUrl = recordCodeReview(recommend);
        // 4. 发送消息通知；日志地址、通知的内容
        pushMessage(logUrl);
    } catch (Exception e) {
        logger.error("openai-code-review error", e);
        throw new RuntimeException("Code review execution failed", e);
    }
}

// GitCommand.java
public String diff() throws IOException, InterruptedException {
    // 获取最新提交哈希
    ProcessBuilder logProcessBuilder = new ProcessBuilder("git", "log", "-1", "--pretty=format:%H");
    logProcessBuilder.directory(new File("."));
    Process logProcess = logProcessBuilder.start();

    try (BufferedReader logReader = new BufferedReader(new InputStreamReader(logProcess.getInputStream()))) {
        String latestCommitHash = logReader.readLine();
        if (latestCommitHash == null) {
            throw new IllegalStateException("No commits found in repository");
        }
        
        // 获取diff
        ProcessBuilder diffProcessBuilder = new ProcessBuilder("git", "diff", latestCommitHash + "^", latestCommitHash);
        diffProcessBuilder.directory(new File("."));
        Process diffProcess = diffProcessBuilder.start();

        try (BufferedReader diffReader = new BufferedReader(new InputStreamReader(diffProcess.getInputStream()))) {
            StringBuilder diffCode = new StringBuilder();
            String line;
            while ((line = diffReader.readLine()) != null) {
                diffCode.append(line).append("\n");
            }

            int exitCode = diffProcess.waitFor();
            if (exitCode != 0) {
                throw new RuntimeException("Failed to get diff, exit code:" + exitCode);
            }
            return diffCode.toString();
        }
    }
}

// GitCommand.java - commitAndPush方法
public String commitAndPush(String recommend) throws Exception {
    try (Git git = Git.cloneRepository()
            .setURI(githubReviewLogUri + ".git")
            .setDirectory(new File("repo"))
            .setCredentialsProvider(new UsernamePasswordCredentialsProvider(githubToken, ""))
            .call()) {

        // 创建分支
        String dateFolderName = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
        File dateFolder = new File("repo/" + dateFolderName);
        if (!dateFolder.exists()) {
            dateFolder.mkdirs();
        }

        // 使用UUID生成文件名
        String fileName = project + "-" + branch + "-" + author + "-" + UUID.randomUUID() + ".md";
        File newFile = new File(dateFolder, fileName);
        try (FileWriter writer = new FileWriter(newFile)) {
            writer.write(recommend);
        }

        // 提交内容
        git.add().addFilepattern(dateFolderName + "/" + fileName).call();
        git.commit().setMessage("add code review new file" + fileName).call();
        git.push().setCredentialsProvider(new UsernamePasswordCredentialsProvider(githubToken, "")).call();

        logger.info("openai-code-review git commit and push done! {}", fileName);
        return githubReviewLogUri + "/blob/master/" + dateFolderName + "/" + fileName;
    }
}

// TemplateMessageDTO.java
public class TemplateMessageDTO {
    // ... 其他代码 ...

    public static void put(Map<String, Map<String, String>> data, TemplateKey key, String value) {
        put(data, key.getCode(), value);
    }

    private static void put(Map<String, Map<String, String>> data, String key, String value) {
        data.put(key, createValueMap(value));
    }

    private static Map<String, String> createValueMap(String value) {
        Map<String, String> valueMap = new HashMap<>();
        valueMap.put("value", value);
        return valueMap;
    }
}
```