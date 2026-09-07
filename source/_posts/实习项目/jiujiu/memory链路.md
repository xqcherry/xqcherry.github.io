---
title: memory链路
date: 2026-06-15 22:33:28
categories:
  - 笔记
  - 实习项目
tags:
  - jiujiu
---
# Memory链路

所以我会把这套设计总结成：

这个 memory 系统本质上是一个 profile 级的长期上下文管理系统，记忆被绑定到当前用户画像，它用异步抽取把聊天里的长期价值信息沉淀成结构化 memo point，再在后续聊天时作为 SYSTEM context 注入。系统有两类记忆，自动记忆和手动记忆，两者共用同一套存储模型，但通过 source 区分来源；同时用激活状态（active）、发生证据（occurrence ）、内容校验、profile 隔离和注入长度限制，保证记忆既能个性化回复，又相对可控、可解释、可删除。



我从三个角度介绍memory链路，记忆怎么产生，怎么保存，怎么在下一轮聊天中发挥作用



首先，系统里有两类记忆：一类是 **AI 自动长期记忆**，一类是**用户手动维护的记忆点**。它们最后都会变成结构化的`memo point`，区别在于来源不一样，**AI 自动记忆**是系统从用户的聊天记录里抽取出来的，比如用户多次提到“对方冷战”“对方在节日里照顾我”，系统会把这些沉淀成长期可复用的信息。**手动记忆**则是用户在自己的memory面板里增删查改的内容，用户可以明确告诉系统哪些记忆点应该被记住。



自动记忆这条链路不是在用户发消息时同步执行的，聊天主链路只负责正常回复，避免影响响应速度。后台会有一个定时的异步任务，扫描开启了 memory 的关系档案，判断用户最近是否已经静默了一段时间，如果用户还在连续聊天，就先不抽取，等对话停下来以后，再把上次处理之后的新用户消息取出来，交给专门的 Memory AI 做抽取



Memory AI 的任务不是总结整段聊天，而是抽取有长期价值的关系事件，当前系统把记忆分成两类：REDFLAG 和 SWEET（雷点和甜蜜事件）,REDFLAG 表示关系里的风险、冲突、边界问题，SWEET 表示温暖、支持、亲密的正向时刻。这样做的好处是，，后续 AI 回复时不仅知道用户“有什么问题”，也知道这段关系里“有什么好的基础”。



抽取结果不会直接无脑入库，。后端还会做一层校验：内容不能太泛，比如“沟通问题”“缺乏安全感”这种没有具体事件的描述会被过滤，也不能包含敏感信息，比如邮箱、手机号、token 之类，同时会校验标题和详情长度。通过这层后端校验，可以减少 AI 抽取出来的低质量长期记忆。



保存的时候，系统会把记忆存成 memo point，每个 memo point 有类型、标题、详情、来源、是否 active、出现次数等字段。来源字段很重要：如果是 AI 抽取的，就是 AI，如果是用户手动创建或编辑的，就是 MANUAL。这样系统在产品和管理上都能区分：哪些是 AI 自动推断出来的，哪些是用户明确确认过的。



除此之外，还设计了一个 occurrence，也就是“发生记录”，如果同一个 memo point 后续又在聊天里出现，系统不会简单重复创建一条新记忆，而是给原来的 memo point 增加一次occurrenc，occurrence 会保存发生时间和相关用户消息片段。这样用户打开记忆详情时，可以看到这个记忆点是从哪些聊天内容里来的，也可以删除某一次 occurrence。这让自动记忆变得更可解释、可追溯。



然后是注入链路。每次用户发起聊天，后端构建 AI 上下文时，会读取当前关系档案下 active 的 memo points。如果 memory 开关是关闭的，就不注入，也不继续产生新记忆。如果开启，就把 redflags 和 sweet moments 分组拼成一段 SYSTEM 上下文，放到模型prompt 里。为了控制 token 成本，系统会限制注入的条数和总字符数，不会把所有历史记忆都塞进去。比如前任、现任、暧昧对象。每个档案有自己的 memory 开关、自己的 memo points、自己的处理游标。这样可以避免不同关系对象之间的记忆串到一起，我觉得这是关系类产品里很重要的边界设计。





 1. 记忆分两类
    代码里长期记忆的核心实体是 UserMemoPointEntity：

  backend/src/main/java/com/example/aichat/memory/UserMemoPointEntity.java:13

  它里面几个字段很关键：

  private MemoPointType type;      // REDFLAG / SWEET
  private MemoPointSource source;  // AI / MANUAL
  private String title;
  private String detail;
  private int occurrenceCount;
  private boolean active;

  MemoPointType 只有两种：

  backend/src/main/java/com/example/aichat/memory/MemoPointType.java:3

  public enum MemoPointType {
      REDFLAG,
      SWEET
  }

  MemoPointSource 也只有两种：

  backend/src/main/java/com/example/aichat/memory/MemoPointSource.java:3

  public enum MemoPointSource {
      AI,
      MANUAL
  }

  所以代码层面，“AI 自动长期记忆”和“用户手动维护记忆点”不是两套表，而是同一套 memo point，通过 source 区分。

  2. 手动记忆怎么实现

    前端 Memory 面板调用 /api/memory 相关接口：

  frontend/app/components/UserMemoryPanel.tsx:173

  比如：

  fetch(`${API_BASE}/api/memory/settings`)
  fetch(`${API_BASE}/api/memory?type=${type}`)
  fetch(`${API_BASE}/api/memory/${id}`)
  fetch(`${API_BASE}/api/memory/${item.id}/occurrences?page=0&size=20`)

  后端入口是 MemoryController：

  backend/src/main/java/com/example/aichat/memory/MemoryController.java:11

  @RestController
  @RequestMapping("/api/memory")
  public class MemoryController

  它提供：

  GET    /api/memory
  POST   /api/memory
  PATCH  /api/memory/{memoryId}
  DELETE /api/memory/{memoryId}
  GET    /api/memory/settings
  PATCH  /api/memory/settings

  真正业务在 MemoryService：

  backend/src/main/java/com/example/aichat/memory/MemoryService.java:62

  用户手动创建时：

  entity.setSource(MemoPointSource.MANUAL);

  更新时也是：

  backend/src/main/java/com/example/aichat/memory/MemoryService.java:95

  existing.setSource(MemoPointSource.MANUAL);

  这就是你面试里说的：“手动记忆是用户显式维护的，source 是 MANUAL”。

  3. 自动记忆不是聊天同步抽取

    自动抽取入口不是聊天接口，而是 MemoPointScheduler：

  backend/src/main/java/com/example/aichat/memory/MemoPointScheduler.java:27

  @Scheduled(fixedDelayString = "${app.weather-score.silent-scan-interval-ms:60000}")
  public void extractMemoPoints() {
      Instant now = Instant.now();
      List<RelationshipProfileEntity> profiles =
          relationshipProfileRepository.findAllByArchivedFalseAndMemoryEnabledTrue();

      for (RelationshipProfileEntity profile : profiles) {
          memoryExtractionService.extractPendingUsersMessagesIfSilent(profile, now);
      }
  }

  这段可以这样理解：

  后台定时扫描所有“未归档且开启 memory 的关系档案”，然后交给 MemoryExtractionService 做抽取。

  所以面试里你可以说：

  “自动记忆不是用户发消息时同步跑的，而是后台定时任务异步抽取，避免影响聊天回复延迟。”

  4. 静默后才抽取

    核心逻辑在：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:128

  public void extractPendingUsersMessagesIfSilent(RelationshipProfileEntity profile, Instant now) {
      if (profile == null || !profile.isMemoryEnabled()) {
          return;
      }

      Instant latestUserMessageAt =
          messageRepository.findMaxCreatedAtByProfileIdAndRole(profile.getId(), MessageRole.USER);
    
      long silentSeconds = Math.max(1, appProperties.getWeatherScore().getSilentSeconds());
    
      if (latestUserMessageAt == null || latestUserMessageAt.isAfter(now.minusSeconds(silentSeconds))) {
          return;
      }

  这里做了两件事：

  第一，profile 没开 memory，直接跳过。

  第二，查这个 profile 下最新一条用户消息时间。如果用户刚刚还在发消息，没有超过静默时间，也跳过。

  所以它不是“每轮聊天后马上抽”，而是等用户停下来再批处理。

  5. 只处理没处理过的新消息

    继续看同一个方法：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:147

  Instant after = profile.getMemoryLastProcessedUserMessageAt();
  List<ChatMessageEntity> pending = loadPendingUserMessages(profile.getId(), after);

  memoryLastProcessedUserMessageAt 就是抽取游标。它记录这个 profile 的用户消息处理到哪了。

  加载逻辑在：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:235

  private List<ChatMessageEntity> loadPendingUserMessages(UUID profileId, Instant after) {
      if (after == null) {
          return messageRepository.findBySessionProfileIdAndRoleOrderByCreatedAtAsc(profileId, MessageRole.USER);
      }
      return messageRepository.findBySessionProfileIdAndRoleAndCreatedAtAfterOrderByCreatedAtAsc(
              profileId,
              MessageRole.USER,
              after
      );
  }

  处理完成后推进游标：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:226

  profile.setMemoryLastProcessedUserMessageAt(checkpoint);
  relationshipProfileRepository.save(profile);

  面试里可以说：

  “它用 profile 上的 last processed cursor 保证增量抽取，不重复扫描全部聊天历史。”

  6. Memory AI 怎么抽取

    构造 prompt 的地方：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:166

  List<UserMemoPointEntity> existing =
      memoPointRepository.findByProfileIdAndActiveTrueOrderByUpdatedAtDesc(profile.getId());

  String userPrompt = buildUserPrompt(existing, pending);

  List<AiDtos.AiMessage> promptMessages = List.of(
      new AiDtos.AiMessage(MessageRole.SYSTEM, promptRuntimeService.systemPrompt(PromptKey.MEMORY_EXTRACTION)),
      new AiDtos.AiMessage(MessageRole.USER, userPrompt)
  );

  AiDtos.AiResponse aiResponse = memoryAiClient.complete(promptMessages);

  这说明抽取时会把两部分都给 Memory AI：

  1. 已有 memo points
  2. 新增的用户消息

  为什么要传已有 memo points？因为 AI 可以判断“这是一个新记忆点”，还是“这是已有记忆点的一次新 occurrence”。

  Memory AI 的调用封装在：

  backend/src/main/java/com/example/aichat/memory/MemoryAiClient.java:32

  它支持 OpenAI chat completions 和 Ark responses 两种 provider，并且是非 stream 的普通 completion。

  7. 后端不会无脑相信 AI

    AI 返回后，代码先解析 JSON：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:174

  String raw = aiResponse.content();
  String json = extractJsonObject(raw);
  if (json == null) {
      auditService.log(profile.getUser().getId(), "MEMO_EXTRACT_PARSE_FAILED", ...);
      return;
  }

  然后取：

  JsonNode redflags = root.path("redflags");
  JsonNode sweets = root.path("sweets");

  真正保存前，还有校验：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:308

  String title = MemoryUtils.normalizeAndCapitalizeFirstLatinLetter(...);
  String detail = MemoryUtils.normalizeAndCapitalizeFirstLatinLetter(...);

  if (title.length() < 2 || title.length() > 120) return null;
  if (detail.length() < 10 || detail.length() > 1000) return null;
  if (MemoryUtils.isSensitive(title) || MemoryUtils.isSensitive(detail)) return null;
  if (isTooGeneric(title, detail)) return null;

  这对应你的话术：

  “AI 抽出来的内容不会直接入库，后端还会做长度、敏感信息、泛化程度校验。”

  敏感信息过滤在：

  backend/src/main/java/com/example/aichat/memory/MemoryUtils.java:48

  return LIKELY_SECRET.matcher(s).find()
      || EMAIL.matcher(s).find()
      || PHONE.matcher(s).find();

  泛化过滤在：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:483

  它会过滤类似“communication issues”“缺乏安全感”这种没有具体动作、场景、角色的抽象描述。

  8. 自动记忆怎么落库

    保存 AI 记忆点的地方：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:308

  entity.setUser(profile.getUser());
  entity.setProfile(profile);
  entity.setType(type);
  entity.setTitle(title);
  entity.setDetail(detail);
  entity.setPointDate(pointDate);
  entity.setSource(MemoPointSource.AI);
  entity.setSourceSessionId(source.getSession().getId());
  entity.setSourceMessageId(source.getId());
  entity.setActive(true);

  这里 source=AI，并且记录了来源 session/message。

  去重靠 content hash：

  String hash = MemoryService.contentHash(type, title, detail);

  UserMemoPointEntity entity = memoPointRepository
      .findByProfileIdAndContentHash(profile.getId(), hash)
      .orElseGet(UserMemoPointEntity::new);

  所以它不会轻易重复创建相同内容。

  9. occurrence 是证据链

    occurrence 的实体是：

  backend/src/main/java/com/example/aichat/memory/UserMemoPointOccurrenceEntity.java:1

  自动抽取保存 occurrence 的地方：

  backend/src/main/java/com/example/aichat/memory/MemoryExtractionService.java:246

  UserMemoPointOccurrenceEntity occurrence = new UserMemoPointOccurrenceEntity();
  occurrence.setMemoPoint(point);
  occurrence.setUser(profile.getUser());
  occurrence.setProfile(profile);
  occurrence.setOccurredAt(occurredAt.get());
  occurrence.setChatContextJson(chatContext.get());
  occurrenceRepository.save(occurrence);

  然后给 memo point 累加次数：

  point.incrementOccurrenceCount(pointWrites);
  memoPointRepository.save(point);

  这就是你说的：

  “如果同一个记忆点后面又出现，不一定新建一条，而是增加 occurrence，并保存这次发生的聊天上下文证据。”

  前端查看 occurrence：

  frontend/app/components/UserMemoryPanel.tsx:240

  fetch(`${API_BASE}/api/memory/${item.id}/occurrences?page=0&size=20`)

  后端接口：

  backend/src/main/java/com/example/aichat/memory/MemoryController.java:40

  @GetMapping("/{memoryId}/occurrences")

  用户还能删除某一次 occurrence：

  backend/src/main/java/com/example/aichat/memory/MemoryService.java:153

  删除后会减少 occurrenceCount。

  10. 下一轮聊天怎么注入

    注入入口在聊天上下文构建：

  backend/src/main/java/com/example/aichat/chat/reply/ChatContextBuilder.java:84

  Optional<String> memorySystem = memoryContextService.buildSystemMemoryMessage(profileId);

  MemoryContextService 里先检查 profile memory 开关：

  backend/src/main/java/com/example/aichat/memory/MemoryContextService.java:31

  RelationshipProfileEntity profile = relationshipProfileRepository.findById(profileId).orElse(null);
  if (profile == null || !profile.isMemoryEnabled()) return Optional.empty();

  然后分别取 redflags 和 sweets：

  List<UserMemoPointEntity> redflags =
      memoPointRepository.findTop50ByProfileIdAndTypeAndActiveTrueOrderByUpdatedAtDesc(
          profileId, MemoPointType.REDFLAG);

  List<UserMemoPointEntity> sweets =
      memoPointRepository.findTop50ByProfileIdAndTypeAndActiveTrueOrderByUpdatedAtDesc(
          profileId, MemoPointType.SWEET);

  拼成 SYSTEM message：

  backend/src/main/java/com/example/aichat/memory/MemoryContextService.java:51

  count = appendPoints(sb, "Relevant redflags", redflags, maxItems, maxChars, count);
  count = appendPoints(sb, "Relevant sweet moments", sweets, maxItems, maxChars, count);

  具体格式：

  backend/src/main/java/com/example/aichat/memory/MemoryContextService.java:83

  String line = "- [" + point.getTitle().trim() + ", " + point.getOccurrenceCount() + " times] "
          + point.getDetail().trim();

  所以注入给模型的大概是：

  Relevant redflags:
  - [Cold war after conflict, 3 times] Partner ignored messages after arguments...

  Relevant sweet moments:
  - [Supported user during work stress, 2 times] Partner comforted user...

  然后 ChatContextBuilder 把它放进 AI messages：

  backend/src/main/java/com/example/aichat/chat/reply/ChatContextBuilder.java:91

  personaSystem.content().ifPresent(text -> out.add(new AiDtos.AiMessage(MessageRole.SYSTEM, text)));
  out.add(new AiDtos.AiMessage(MessageRole.SYSTEM, ReplyLanguageSupport.strictReplyInstruction(preferredLanguage)));

  if (!personaSystem.memoryRendered()) {
      memorySystem.ifPresent(text -> out.add(new AiDtos.AiMessage(MessageRole.SYSTEM, text)));
  }

  这里还有个细节：如果主 persona prompt 里包含 {{MEMORY}}，就把 memory 渲染到 persona prompt 里；否则单独作为一条 SYSTEM
  message 注入。

  11. profile 级隔离

    很多方法都不是按 userId 直接读记忆，而是先拿 active relationship profile：

  backend/src/main/java/com/example/aichat/memory/MemoryService.java:54

  RelationshipProfileEntity profile =
      relationshipProfileAccessService.getRequiredActiveProfile(userId);

  return memoPointRepository.listForProfile(profile.getId(), type)

  自动抽取也是按 profile：

  profile.getId()

  注入也是按 profile：

  buildSystemMemoryMessage(profileId)

  这就是你面试要强调的：

  “记忆不是简单用户级，而是关系档案级。一个用户有多个关系对象时，记忆不会串。”

  最后你可以这样把代码层面讲成一句完整的话
  代码里 Memory 链路分成三块：MemoryService/MemoryController 处理用户手动维护，创建或编辑时把 memo point 标成 MANUAL；
  MemoPointScheduler + MemoryExtractionService 处理 AI 自动抽取，定时扫描开启 memory 的 profile，等用户静默后增量读取未
  处理的 USER 消息，调用 MemoryAiClient 抽取 REDFLAG/SWEET，通过后端校验后保存为 AI 来源的 memo point，并记录 occurrence
  证据；最后 ChatContextBuilder 在每次聊天前调用 MemoryContextService，按当前 profile 拉取 active memo points，限制数量
  和字符数后注入 SYSTEM prompt，让下一轮回复具备长期上下文。