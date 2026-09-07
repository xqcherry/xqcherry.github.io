---
title: 多agent协作
date: 2026-06-15 15:29:30
categories:
  - 笔记
  - 实习项目
tags:
  - jiujiu
---
# 多agent协作



项目不是简单的用户发一句话，后端调一次大模型返回，而是实现了一个多agent子能力编排系统，主聊天agent负责最终回复，围绕主回复前后，还有一组专门的小agent负责意图判断，风险检测，标题生成，追问生成，记忆抽取，测评报告和关系天气分析，每个agent子模块都有独立的prompt，配置和计费channel，职责边界比较清晰



主链路是聊天agent，用户发信息后，后端会先保存用户信息，然后进入会话编排器，编排器不是马上调用主模型，而是做一些前置判断，比如意图检测，意图检测ai作用是判断用户是不是明确想做关系测评，如果识别到“我想做测试‘’之类的意图，系统就不走普通聊天回复，而是直接返回测评 CTA 卡片，这样测评入口不是靠主模型随口生成，而是由一个专门的分类能力控制。边界比较清晰，可自主监控。



如果没有命中测评意图，系统才进入主聊天回复。主聊天 AI 调用前，会先由专门的上下文构建函数构建上下文，把最近聊天历史、长期 memory、RAG 知识库、relationship weather context、附件上下文等拼好，调用主聊天模型生成回复



同时，PUA detection AI 会异步跑，它不是等主回复结束后才开始，而是在主聊天生成前就基于当前 session 的用户消息启动异步检测。任务是判断用户描述的对话关系中是否存在pua行为，因为这是辅助安全能力，所以它不会阻塞主回复。如果超时或失败，会自动降级，不影响正常聊天。如果检测到风险，系统会落库一条 alert，并通过 SSE 给前端发PUA 风险提示、文字卡片和图片消息



 主聊天回复完成后，还有几个后置 AI 能力。第一是 title generation AI。如果这是新会话，标题还是 “New Chat” 这种placeholder，并且当前会话刚好只有一轮用户消息和一轮助手回复，系统会调用标题生成 AI，根据首条用户消息和首条 AI 回复生成一个简短会话标题。这个模块和主聊天模型分开，是因为标题生成是一个小型结构化任务，不需要让主聊天模型承担。

第二是 followup question AI。它会根据当前对话上下文和刚生成的 assistant reply，生成 2 到 3 个用户可能继续追问的问题。这个能力服务的是前端交互体验，让用户可以点推荐问题继续聊。它要求模型返回 JSON 数组，后端再做解析、去重、长度裁剪。如果返回的问题太少或者解析失败，就不会强行展示。

第三是 memory extraction AI。它不在主聊天请求里同步执行，而是后台定时任务在用户静默后处理。它读取未处理过的用户消息，把里面有长期价值的关系事件抽成 memo point，比如 redflag 或 sweet moment。这个 AI 的目标不是回复用户，而是沉淀长期上下文，下一轮聊天时再通过 memory context 注入给主聊天 AI。

然后是 assessment 相关 AI。关系测评本身不是纯聊天，它有一套测评输入、规则评分和 AI 报告生成链路。用户完成测评后，系统会先基于规则引擎算出分数和维度，比如 security、trust、intimacy、communication 等。然后 Assessment Stream AI 根据这些结构化结果生成一份自然语言测评报告，通过 SSE 流式返回。后续还会生成并保存 report card，供用户查看历史报告。

Weather 相关 AI 是另一个长期状态模块。测评完成后，测评分数会初始化或更新 relationship weather score。之后用户继续聊天，后台会在用户静默后分析新增聊天内容，调用 Weather AI 计算每个维度的增量，比如安全感、信任、沟通、亲密度今天是上升还是下降。这个结果会落到 weather score 表里，并且下一轮聊天时作为 RELATIONSHIP_WEATHER_CONTEXT 注入主聊天 AI，让主回复知道当前关系状态的大致趋势。



• 我会按“主聊天编排器先看、再看各 AI 子模块”的顺序，把这条链路对应到具体类和关键方法。重点讲代码怎么串起来，而不是只列文
  件。

• 代码层面可以从一个核心入口讲起：ChatReplyOrchestrator 是主聊天 AI 编排器。

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:21

  它里面注入了很多子模块：

  private final ChatReplyPersistenceService persistenceService;
  private final ChatContextBuilder chatContextBuilder;
  private final ChatReplyAiExecutor aiExecutor;
  private final IntentDetectionService intentDetectionService;
  private final PuaDetectionService puaDetectionService;
  private final CreditService creditService;
  private final AuditService auditService;
  private final TitleGenerationService titleGenerationService;

  这就说明，主聊天不是一个简单的 callAI()，而是由 orchestrator 把多个能力串起来。

  1. 主聊天回复从 Orchestrator 开始
    核心方法是：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:57

  public ChatReplyOutcome reply(ChatReplyCommand command)

  第一步先查 session 和最新用户消息：

  ChatReplyRequestContext requestContext = persistenceService.requireReplyContext(
          command.userId(), command.sessionId(), command.userMessageId());

  ChatSessionEntity session = requestContext.session();
  ChatMessageEntity latestUser = requestContext.latestUserMessage();

  也就是说，它先保证这次回复对应的是合法 session、合法 user message。

  2. Intent Detection AI：前置分流
    接着最先调用的是 intent detection：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:62

  IntentDetectionService.IntentDecision intentDecision = detectIntent(command, session, latestUser);

  如果命中测评意图：

  if (intentDecision.hit()) {
      ChatMessageEntity ctaMessage = persistenceService.saveAssessmentCtaMessage(session);
      ...
      command.output().onAssessmentCta(...);
      command.output().onCompleted();
      return new ChatReplyOutcome(...);
  }

  这段很关键。它说明：

  如果用户明确要做测评，系统不会继续调用主聊天 AI，而是直接返回 assessment CTA。

  IntentDetectionService 的具体逻辑在：

  backend/src/main/java/com/example/aichat/chat/IntentDetectionService.java:15

  它先做规则 fastpath：

  if (explicitAssessmentIntentMatch(normalizedText)) {
      return IntentDecision.hit("rule-fastpath");
  }

  如果规则没命中，再调用 Intent AI：

  List<AiDtos.AiMessage> messages = List.of(
      new AiDtos.AiMessage(MessageRole.SYSTEM, promptRuntimeService.systemPrompt(PromptKey.INTENT_DETECTION)),
      new AiDtos.AiMessage(MessageRole.USER, normalizedText)
  );

  AiDtos.AiResponse aiResponse = intentAiClient.complete(messages);

  然后解析 JSON：

  IntentCode parsed = parseIntent(aiResponse.content());

  所以代码实现上，Intent AI 是一个 前置分类器。

  3. PUA Detection AI：异步并行检测
    如果没有命中 intent，主链路继续。接下来它启动 PUA 检测：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:92

  List<ChatMessageEntity> sessionMessages = persistenceService.listSessionMessages(session);

  CompletableFuture<PuaDetectionService.PuaDecision> puaFuture = puaDetectionService.detectAsync(
          command.userId(),
          session.getProfile().getId(),
          session.getId(),
          latestUser.getId(),
          latestUser.getCreatedAt(),
          sessionMessages
  );

  注意这里是 CompletableFuture。也就是说 PUA 检测和主聊天回复是并行的，不是同步阻塞在前面。

  PuaDetectionService.detectAsync 里：

  backend/src/main/java/com/example/aichat/chat/PuaDetectionService.java:46

  return CompletableFuture.supplyAsync(() -> detect(...))
          .orTimeout(Math.max(1000, cfg.getTimeoutMs()), TimeUnit.MILLISECONDS)
          .exceptionally(ex -> {
              return PuaDecision.insufficient("pua-detection-timeout-or-error");
          });

  如果超时或失败，降级为 INSUFFICIENT，不影响主聊天。

  真正调用 PUA AI 的地方：

  AiDtos.AiResponse aiResponse = puaAiClient.complete(List.of(
      new AiDtos.AiMessage(MessageRole.SYSTEM, promptRuntimeService.systemPrompt(PromptKey.PUA_DETECTION)),
      new AiDtos.AiMessage(MessageRole.USER, userPrompt.toString())
  ));

  它会要求模型返回 JSON：

  {"flag":"PUA|NO_PUA|INSUFFICIENT","reason":"...","keywords":[...],"advice":"..."}

  如果结果是 PUA，会落库 alert：

  PuaDetectionAlertEntity alert = new PuaDetectionAlertEntity();
  alert.setUserId(userId);
  alert.setProfileId(profileId);
  alert.setSessionId(sessionId);
  alert.setTriggerMessageId(triggerMessageId);
  alert.setLocalDate(localDate);
  alert.setReason(reason);
  puaDetectionAlertRepository.save(alert);

  然后回到 orchestrator，主回复完成后等待 PUA 结果：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:142

  PuaDetectionService.PuaDecision puaDecision = waitPuaDecision(session.getId(), puaFuture);

  如果是 PUA，就通过 SSE 发前端事件，并保存 PUA 消息：

  command.output().onPuaAlert(...);
  ChatMessageEntity puaMessage = persistenceService.savePuaHintMessage(session, puaDecision.rawJson());
  ChatMessageEntity puaImageMessage = persistenceService.savePuaImageMessage(session);

  所以 PUA AI 是 并行安全检测能力。

  4. 主聊天 AI：真正生成回复
    PUA future 启动后，才检查积分并构建上下文：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:105

  creditService.requireAvailableCredits(command.userId());

  List<AiDtos.AiMessage> context = chatContextBuilder.build(
          session.getProfile().getId(),
          session,
          command.attachmentContext(),
          command.attachmentImageUrls()
  );

  ChatContextBuilder 会把多种上下文拼起来：

  backend/src/main/java/com/example/aichat/chat/reply/ChatContextBuilder.java:68

  里面包括：

  memoryContextService.buildSystemMemoryMessage(profileId);
  buildWeatherSystemMessage(profileId);
  ragRetrievalService.retrieveTopK(latestUserQuestion);
  ragPromptBuilder.buildSystemRagBlock(hits);

  所以主聊天 AI 调用前，已经拿到了：

  persona prompt
  language instruction
  memory context
  weather context
  RAG context
  attachment context
  recent chat messages

  真正执行 AI 的地方：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:112

  ChatReplyAiExecutionResult aiResult = aiExecutor.execute(new ChatReplyAiExecutionRequest(
          context,
          command.deliveryMode(),
          command.output()::onChunk
  ));

  ChatReplyAiExecutor 根据 delivery mode 选择 adapter：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyAiExecutor.java:9

  return adapters.stream()
          .filter(adapter -> adapter.supports(request.deliveryMode()))
          .findFirst()
          .orElseThrow(...)
          .execute(request);

  也就是说主聊天 AI 支持 streaming 和 non-streaming 两种执行方式。

  主回复生成后保存 assistant message：

  ChatMessageEntity assistant = persistenceService.saveAssistantMessage(
          session,
          aiResult.finalContent(),
          aiResult.modelName(),
          aiResult.usage()
  );

  并按 CHAT_ASSISTANT_REPLY 计费：

  creditService.charge(new CreditChargeRequest(
      command.userId(),
      session.getProfile().getId(),
      session.getId(),
      assistant.getId(),
      CreditSourceType.CHAT_ASSISTANT_REPLY,
      ...
      CreditBillingChannel.CHAT,
      aiResult.provider(),
      aiResult.modelName(),
      aiResult.usage(),
      null
  ));

  5. Title Generation AI：首轮后生成标题
    主回复保存后，orchestrator 会尝试生成会话标题：

  backend/src/main/java/com/example/aichat/chat/reply/ChatReplyOrchestrator.java:134

  Optional<String> generatedTitle = maybeGenerateSessionTitle(session, latestUser, assistant);
  generatedTitle.ifPresent(title -> command.output().onSessionTitle(session.getId(), title));

  触发条件在 maybeGenerateSessionTitle：

  if (!titleGenerationService.isPlaceholderTitle(session.getTitle())) {
      return Optional.empty();
  }

  long userCount = persistenceService.countBySessionAndRole(session, MessageRole.USER);
  long assistantCount = persistenceService.countBySessionAndRole(session, MessageRole.ASSISTANT);
  if (userCount != 1 || assistantCount != 1) {
      return Optional.empty();
  }

  也就是说，只有第一轮对话结束后，且标题还是默认标题时，才生成标题。

  TitleGenerationService 里调用 Title AI：

  backend/src/main/java/com/example/aichat/title/TitleGenerationService.java:26

  AiDtos.AiResponse aiResponse = titleAiClient.complete(messages);

  然后清洗标题：

  String cleaned = cleanTitle(raw, maxChars);

  再保存到 session：

  persistenceService.updateSessionTitle(session, generated.get());

  所以 Title AI 是 后置的小任务生成模型。

  6. Followup Question AI：生成推荐追问
    Followup 不在 ChatReplyOrchestrator 里直接调用，而是在 ChatService 里处理 SSE 主回复后事件。

  搜索结果显示入口在：

  backend/src/main/java/com/example/aichat/chat/ChatService.java:181

  它调用：

  followupQuestionService.generate(...)

  FollowupQuestionService 的核心方法：

  backend/src/main/java/com/example/aichat/followup/FollowupQuestionService.java:47

  public List<String> generate(..., List<AiDtos.AiMessage> contextMessages, String assistantReply)

  它会把当前上下文和 assistant reply 一起给 Followup AI：

  messages.add(new AiDtos.AiMessage(MessageRole.SYSTEM,
  promptRuntimeService.systemPrompt(PromptKey.FOLLOWUP_GENERATION)));
  messages.addAll(contextMessages);
  messages.add(new AiDtos.AiMessage(MessageRole.ASSISTANT, normalizedAssistantReply));
  messages.add(new AiDtos.AiMessage(MessageRole.USER, userPrompt));

  然后调用：

  AiDtos.AiResponse aiResponse = followupAiClient.complete(messages);

  它要求返回 JSON array。后端会解析并清洗：

  List<String> parsed = parseAndClean(raw, maxQuestions);

  清洗逻辑包括：

  去空
  去重
  限制单个问题最长 120 字符
  最多 maxQuestions 条

  所以 Followup AI 是 提升对话连续性的推荐问题生成器。

  7. Memory Extraction AI：后台异步长期记忆
    Memory AI 不在主聊天同步执行，它在 MemoryExtractionService 里：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:128

  后台 scheduler 会调用：

  extractPendingUsersMessagesIfSilent(profile, now)

  核心调用：

  AiDtos.AiResponse aiResponse = memoryAiClient.complete(promptMessages);

  它处理的是用户静默后的 pending user messages，抽取 redflags / sweets，保存成 memo point。下一轮聊天时由
  MemoryContextService 注入主聊天上下文。

  所以 Memory AI 是 后台长期状态抽取模型，不是实时回复模型。

  8. Assessment AI：测评报告生成
    测评链路在 AssessmentService。

  用户开始测评时：

  backend/src/main/java/com/example/aichat/assessment/AssessmentService.java:74

  public AssessmentDtos.StartResponse start(UUID userId, AssessmentDtos.StartRequest request)

  它会把测评表单内容构造成一个 SYSTEM context message：

  context.setRole(MessageRole.SYSTEM);
  context.setContent(buildContextSummary(request, preferredLanguage));

  真正生成测评报告时：

  backend/src/main/java/com/example/aichat/assessment/AssessmentService.java:105

  public SseEmitter stream(UUID userId, UUID sessionId, UUID contextMessageId)

  里面先用规则引擎预打分：

  Optional<AssessmentRuleEngine.Output> previewScore = assessmentReportService.previewScore(ctx.getContent());

  然后调用主 AiService.stream 生成自然语言报告：

  AiDtos.AiResponse aiResponse = aiService.stream(new AiDtos.AiRequest("", messages), (delta) -> {
      safeSend(emitter, ..., "chunk", new ChatDtos.ChunkEvent(delta), ...);
  });

  注意这里用的是主 AiService，但 prompt 是 ASSESSMENT_STREAM，任务是测评报告，不是普通聊天。

  测评流结束后，会保存 assistant message，并调用：

  assessmentReportService.generateAndStore(...)

  AssessmentReportService 会保存结构化报告，并触发 weather score 更新：

  backend/src/main/java/com/example/aichat/assessment/AssessmentReportService.java:83

  weatherScoreService.onAssessmentScored(userId, profileId, saved.getCreatedAt(), scored.totalScore(),
  scored.dimensions(), scored.weatherType());

  所以 Assessment AI 是 测评报告生成模型，结合规则引擎一起工作。

  9. Weather AI：关系状态动态评分
    Weather 有两种来源：

  第一种是测评完成后，直接用 assessment score 初始化或更新 weather score：

  backend/src/main/java/com/example/aichat/weather/WeatherScoreService.java:120

  public void onAssessmentScored(...)

  第二种是聊天后异步分析用户消息：

  backend/src/main/java/com/example/aichat/weather/WeatherScoreService.java:160

  public void analyzePendingChatsIfSilent(RelationshipProfileEntity profile, Instant now)

  它和 Memory 类似，也会等用户静默，读取 pending user messages，然后调用 Weather AI：

  backend/src/main/java/com/example/aichat/weather/WeatherScoreService.java:528

  AiDtos.AiResponse aiResponse = weatherAiClient.complete(List.of(
      new AiDtos.AiMessage(MessageRole.SYSTEM, promptRuntimeService.systemPrompt(PromptKey.WEATHER_SCORING)),
      new AiDtos.AiMessage(MessageRole.USER, userPrompt)
  ));

  Weather AI 返回的是分数 delta：

  totalDelta
  dimensions.SECURITY / TRUST / INTIMACY / COMMUNICATION / COMPATIBILITY / RESILIENCE
  reason

  后端解析后，应用到当前 weather score 上：

  ScoreSnapshot next = current.apply(result.delta(), maxDelta);

  并保存 score event。

  下一轮聊天时，ChatContextBuilder 会读取最新 weather：

  WeatherScoreDtos.LatestScoreResponse latest = weatherScoreService.latestScoreForProfile(profileId);

  然后拼成：

  RELATIONSHIP_WEATHER_CONTEXT
  totalScore=...
  weatherType=...
  dimensions.security=...
  ...

  所以 Weather AI 是 长期关系状态评分模型，它的结果会反过来影响主聊天 AI 的上下文。

  10. 多套 AI 配置，不是一套模型通吃
    代码上也能看到每个 AI 子模块都有独立配置。

  AppProperties 里有：

  backend/src/main/java/com/example/aichat/config/AppProperties.java:15

  private final Ai ai = new Ai();
  private final MemoryAi memoryAi = new MemoryAi();
  private final PuaAi puaAi = new PuaAi();
  private final FollowupAi followupAi = new FollowupAi();
  private final IntentAi intentAi = new IntentAi();
  private final TitleAi titleAi = new TitleAi();
  private final WeatherAi weatherAi = new WeatherAi();

  分别对应：

  主聊天 AI
  Memory extraction AI
  PUA detection AI
  Followup question AI
  Intent detection AI
  Title generation AI
  Weather scoring AI

  每个模块也有自己的 ConfigService：

  MemoryAiConfigService
  WeatherAiConfigService
  PuaAiConfigService
  TitleAiConfigService
  IntentAiConfigService
  FollowupAiConfigService

  所以面试时可以明确说：

  “代码上不是一个 AI client 到处复用配置，而是按能力拆了多个 AI config service 和 client。不同任务可以配置不同
  provider、baseUrl、apiKey、modelName。”

  11. 统一底层调用方式
    虽然上层拆成很多 AI 子模块，但底层调用模式类似：

  ConfigService.resolve()
  构造 system/user messages
  调用 XxxAiClient.complete() 或 AiService.stream()
  解析 JSON 或文本
  计费
  落库或发 SSE 事件
  失败时降级

  主聊天和 assessment stream 主要用流式；intent、title、followup、memory、PUA、weather 多数是非流式 completion。

  你可以这样总结代码实现
  代码上，这个项目用 ChatReplyOrchestrator 编排主聊天生命周期。它先用 IntentDetectionService 做测评意图分流；再异步启动
  PuaDetectionService 做风险检测；然后通过 ChatContextBuilder 聚合 memory、RAG、weather、附件和历史消息，交给
  ChatReplyAiExecutor 调主聊天 AI 流式生成；回复落库后调用 TitleGenerationService 生成首轮标题，并等待 PUA 结果发送风险
  事件。主链路之外，FollowupQuestionService 生成推荐追问，MemoryExtractionService 后台抽取长期记忆，AssessmentService 和
  AssessmentReportService 负责测评报告生成，WeatherScoreService 用 Weather AI 分析聊天和测评结果，维护关系天气分。整体就
  是多个 AI 子模块围绕主聊天前置、并行、后置和后台异步协同。



