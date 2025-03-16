# 游戏好友系统技术设计文档

## 目录

- [1. Redis 数据结构设计](#1-Redis-数据结构设计)
- [2. FriendService 好友服务实现](#2-FriendService-好友服务实现)

## 1. Redis 数据结构设计

```go
const (
	// User Center 公共区域 - 用户基础信息
	KEY_USER_CENTER  = "uc:user:{uid}"   // Hash - 用户基本信息(等级、头像、名称等)
	KEY_USER_STATUS  = "uc:status:{uid}" // Hash - 用户状态信息(在线状态、当前游戏等)
	KEY_ONLINE_USERS = "uc:online"       // Set - 在线用户集合

	// 好友系统
	KEY_FRIEND_HASH = "friend:hash:{uid}"  // Hash - 好友关系(key:好友ID, value:json包含时间和好友度)
	KEY_FRIEND_REQ  = "friend:req:{uid}"   // Hash - 好友申请(key:申请者ID, value:json包含时间和消息)
	KEY_BLACKLIST   = "friend:black:{uid}" // Hash - 黑名单(key:目标ID, value:json包含时间和原因)

	// 好友度相关常量
	INTIMACY_ADD_CHAT = 1        // 聊天增加好友度
	INTIMACY_ADD_GAME = 2        // 一起游戏增加好友度
	INTIMACY_ADD_GIFT = 5        // 送礼物增加好友度
	INTIMACY_MAX_PER_DAY = 10    // 每日最大增加好友度
	
	// 状态更新通知的 Redis Stream key
	KEY_STATUS_NOTIFY = "notify:status"    // Stream - 状态更新通知队列
	KEY_INTIMACY_DAILY = "friend:intimacy:daily:{uid}:{fid}"  // String - 每日好友度计数
)

const (
	// 添加好友关系的 Lua 脚本
	ADD_FRIEND_SCRIPT = `
        local userID = KEYS[1]
        local friendID = KEYS[2]
        local friendInfo = ARGV[1]
        
        -- 检查是否已经是好友
        local existingFriend = redis.call('HGET', 'friend:hash:'..userID, friendID)
        if existingFriend then
            return {err = "already friends"}
        end
        
        -- 检查是否在黑名单中
        local isBlocked = redis.call('HGET', 'friend:black:'..userID, friendID)
        if isBlocked then
            return {err = "user is blocked"}
        end
        
        -- 原子性地添加双向好友关系
        redis.call('HSET', 'friend:hash:'..userID, friendID, friendInfo)
        redis.call('HSET', 'friend:hash:'..friendID, userID, friendInfo)
        
        return {ok = 1}
    `

	// 更新用户状态的 Lua 脚本
	UPDATE_STATUS_SCRIPT = `
        local userID = KEYS[1]
        local status = ARGV[1]
        local customStatus = ARGV[2]
        local currentGame = ARGV[3]
        
        -- 更新状态
        redis.call('HMSET', 'uc:status:'..userID, 
            'status', status,
            'custom_status', customStatus,
            'current_game', currentGame,
            'updated_at', redis.call('TIME')[1]
        )
        
        -- 更新在线用户集合
        if status == '1' then
            redis.call('SADD', 'uc:online', userID)
        else
            redis.call('SREM', 'uc:online', userID)
        end
        
        -- 获取好友列表用于通知
        return redis.call('HKEYS', 'friend:hash:'..userID)
    `

	// 添加到黑名单的 Lua 脚本
	ADD_BLACKLIST_SCRIPT = `
        local userID = KEYS[1]
        local targetID = KEYS[2]
        local reason = ARGV[1]
        
        -- 检查并删除好友关系
        local isFriend = redis.call('HEXISTS', 'friend:hash:'..userID, targetID)
        if isFriend == 1 then
            -- 删除双向好友关系
            redis.call('HDEL', 'friend:hash:'..userID, targetID)
            redis.call('HDEL', 'friend:hash:'..targetID, userID)
        end
        
        -- 添加到黑名单
        local blacklistInfo = string.format('{"target_id":"%s","reason":"%s","create_time":%d}',
            targetID, reason, redis.call('TIME')[1])
        redis.call('HSET', 'friend:black:'..userID, targetID, blacklistInfo)
        
        return {ok = 1}
    `

	// 处理好友请求的 Lua 脚本
	HANDLE_REQUEST_SCRIPT = `
        local receiverID = KEYS[1]
        local senderID = KEYS[2]
        local status = ARGV[1]
        local friendInfo = ARGV[2]
        
        -- 检查请求是否存在
        local req = redis.call('HGET', 'friend:req:'..receiverID, senderID)
        if not req then
            return {err = "request not found"}
        end
        
        -- 如果接受请求
        if status == '1' then
            -- 检查是否已经是好友
            local existingFriend = redis.call('HGET', 'friend:hash:'..receiverID, senderID)
            if existingFriend then
                return {err = "already friends"}
            end
            
            -- 检查是否在黑名单中
            local isBlocked = redis.call('HGET', 'friend:black:'..receiverID, senderID)
            if isBlocked then
                return {err = "user is blocked"}
            end
            
            -- 建立双向好友关系
            redis.call('HSET', 'friend:hash:'..receiverID, senderID, friendInfo)
            redis.call('HSET', 'friend:hash:'..senderID, receiverID, friendInfo)
        end
        
        -- 删除请求
        redis.call('HDEL', 'friend:req:'..receiverID, senderID)
        
        return {ok = 1, status = status}
    `
)

// User Center 数据结构
type UserCenterInfo struct {
	UserID     string `json:"user_id"`
	Username   string `json:"username"`
	Level      int32  `json:"level"`
	Avatar     string `json:"avatar"`
	Title      string `json:"title"`
	VipLevel   int32  `json:"vip_level"`
	LastActive int64  `json:"last_active"`
}

// 好友关系数据
type FriendInfo struct {
	FriendID   string `json:"friend_id"`
	Intimacy   int32  `json:"intimacy"` // 好友度
	CreateTime int64  `json:"create_time"`
}

// 好友申请数据
type FriendRequest struct {
	SenderID   string `json:"sender_id"`
	Message    string `json:"message"`
	CreateTime int64  `json:"create_time"`
}

// 黑名单数据
type BlacklistInfo struct {
	TargetID   string `json:"target_id"`
	Reason     string `json:"reason"`
	CreateTime int64  `json:"create_time"`
}

// StatusManager 状态管理器
type StatusManager struct {
	svc    *FriendService
	redis  *redis.Client
	logger *zap.Logger
	
	// 通知重试配置
	maxRetries int
	retryDelay time.Duration
}

// NewStatusManager 创建状态管理器
func NewStatusManager(svc *FriendService, redis *redis.Client, logger *zap.Logger) *StatusManager {
	return &StatusManager{
		svc:        svc,
		redis:      redis,
		logger:     logger,
		maxRetries: 3,
		retryDelay: time.Second * 5,
	}
}
```

## 2. FriendService 好友服务实现

```go
// FriendService 好友服务实现
type FriendService struct {
	redis  *redis.Client
	logger *zap.Logger

	// Lua 脚本
	addFriendScript        *redis.Script
	updateStatusScript     *redis.Script
	addBlacklistScript    *redis.Script
	handleRequestScript   *redis.Script
}

const (
	// 好友请求状态
	REQUEST_STATUS_PENDING  = 0  // 待处理
	REQUEST_STATUS_ACCEPTED = 1  // 已接受
	REQUEST_STATUS_REJECTED = 2  // 已拒绝
	
	// 好友请求过期时间 (30天)
	REQUEST_EXPIRE_TIME = 60 * 60 * 24 * 30
)

// NewFriendService 创建好友服务实例
func NewFriendService(redis *redis.Client, logger *zap.Logger) *FriendService {
	return &FriendService{
		redis:  redis,
		logger: logger,

		// 初始化 Lua 脚本
		addFriendScript:     redis.NewScript(ADD_FRIEND_SCRIPT),
		updateStatusScript:  redis.NewScript(UPDATE_STATUS_SCRIPT),
		addBlacklistScript:  redis.NewScript(ADD_BLACKLIST_SCRIPT),
		handleRequestScript: redis.NewScript(HANDLE_REQUEST_SCRIPT),
	}
}

// AddFriend 直接添加好友关系(仅用于特殊场景)
// 警告: 此方法跳过了正常的好友请求流程,仅用于:
// 1. 导入好友数据
// 2. 系统自动添加好友
// 3. 管理员操作
// 正常的社交流程应使用 CreateFriendRequest 和 HandleFriendRequest
func (s *FriendService) AddFriend(ctx context.Context, userID, friendID int64) error {
	// 创建好友信息
	info := &FriendInfo{
		FriendID:   strconv.FormatInt(friendID, 10),
		CreateTime: time.Now().Unix(),
		Intimacy:   0,
	}

	// 序列化好友信息
	friendInfo, err := json.Marshal(info)
	if err != nil {
		return fmt.Errorf("failed to marshal friend info: %w", err)
	}

	// 执行 Lua 脚本
	result, err := s.addFriendScript.Run(ctx, s.redis,
		[]string{strconv.FormatInt(userID, 10), strconv.FormatInt(friendID, 10)},
		string(friendInfo),
	).Result()

	if err != nil {
		return fmt.Errorf("failed to execute add friend script: %w", err)
	}

	// 处理结果
	res, ok := result.(map[string]interface{})
	if !ok {
		return errors.New("invalid script result")
	}

	if errMsg, exists := res["err"]; exists {
		return errors.New(errMsg.(string))
	}

	return nil
}

// GetFriendList 获取好友列表
func (s *FriendService) GetFriendList(ctx context.Context, userID string) ([]*FriendInfo, error) {
	// 获取好友 ID 列表
	key := fmt.Sprintf(KEY_FRIEND_HASH, userID)
	result, err := s.redis.HGetAll(ctx, key).Result()
	if err != nil {
		return nil, err
	}

	// 解析好友信息
	friends := make([]*FriendInfo, 0, len(result))
	for _, v := range result {
		var info FriendInfo
		if err := json.Unmarshal([]byte(v), &info); err != nil {
			s.logger.Error("failed to unmarshal friend info",
				zap.Error(err),
				zap.String("data", v))
			continue
		}
		friends = append(friends, &info)
	}

	return friends, nil
}

// GetFriendRequests 获取好友请求列表
func (s *FriendService) GetFriendRequests(ctx context.Context, userID int64) ([]*FriendRequest, error) {
	key := fmt.Sprintf(KEY_FRIEND_REQ, userID)
	result, err := s.redis.HGetAll(ctx, key).Result()
	if err != nil {
		return nil, fmt.Errorf("failed to get friend requests: %w", err)
	}

	requests := make([]*FriendRequest, 0, len(result))
	for _, v := range result {
		var req FriendRequest
		if err := json.Unmarshal([]byte(v), &req); err != nil {
			s.logger.Error("failed to unmarshal friend request",
				zap.Error(err),
				zap.String("data", v))
			continue
		}
		requests = append(requests, &req)
	}

	// 按创建时间排序
	sort.Slice(requests, func(i, j int) bool {
		return requests[i].CreateTime > requests[j].CreateTime
	})

	return requests, nil
}

// DeleteFriendRequest 删除好友请求
func (s *FriendService) DeleteFriendRequest(ctx context.Context, userID, senderID int64) error {
	key := fmt.Sprintf(KEY_FRIEND_REQ, userID)
	return s.redis.HDel(ctx, key, fmt.Sprint(senderID)).Err()
}

// AddToBlacklist 添加用户到黑名单
func (s *FriendService) AddToBlacklist(ctx context.Context, userID, targetID string, reason string) error {
	// 执行 Lua 脚本
	result, err := s.addBlacklistScript.Run(ctx, s.redis,
		[]string{userID, targetID},
		reason,
	).Result()

	if err != nil {
		return err
	}

	// 处理结果
	res, ok := result.(map[string]interface{})
	if !ok {
		return errors.New("invalid script result")
	}

	if errMsg, exists := res["err"]; exists {
		return errors.New(errMsg.(string))
	}

	return nil
}

// RemoveFromBlacklist 从黑名单移除用户
func (s *FriendService) RemoveFromBlacklist(ctx context.Context, userID, targetID string) error {
	key := fmt.Sprintf(KEY_BLACKLIST, userID)
	return s.redis.HDel(ctx, key, targetID).Err()
}

// GetBlacklist 获取黑名单列表
func (s *FriendService) GetBlacklist(ctx context.Context, userID string) ([]*BlacklistInfo, error) {
	key := fmt.Sprintf(KEY_BLACKLIST, userID)
	result, err := s.redis.HGetAll(ctx, key).Result()
	if err != nil {
		return nil, err
	}

	blacklist := make([]*BlacklistInfo, 0, len(result))
	for _, v := range result {
		var info BlacklistInfo
		if err := json.Unmarshal([]byte(v), &info); err != nil {
			s.logger.Error("failed to unmarshal blacklist info",
				zap.Error(err),
				zap.String("data", v))
			continue
		}
		blacklist = append(blacklist, &info)
	}

	return blacklist, nil
}

// UpdateStatus 更新用户状态并通知好友
func (s *FriendService) UpdateStatus(ctx context.Context, userID int64, status int32, customStatus, currentGame string) error {
	// 使用定义好的 KEY_USER_STATUS
	statusKey := fmt.Sprintf(KEY_USER_STATUS, userID)

	// 执行 Lua 脚本更新状态
	result, err := s.updateStatusScript.Run(ctx, s.redis,
		[]string{statusKey, KEY_ONLINE_USERS},
		status, customStatus, currentGame,
	).Result()

	if err != nil {
		return fmt.Errorf("failed to update status: %w", err)
	}

	// 获取好友列表进行通知
	friendIDs, ok := result.([]interface{})
	if !ok {
		return errors.New("invalid script result type")
	}

	// 创建状态通知
	notify := &StatusNotification{
		UserID:      userID,
		Status:      status,
		CustomStatus: customStatus,
		CurrentGame: currentGame,
		UpdatedAt:   time.Now().Unix(),
	}

	// 序列化通知
	notifyData, err := json.Marshal(notify)
	if err != nil {
		return fmt.Errorf("failed to marshal notification: %w", err)
	}

	// 将通知添加到 Redis Stream
	for _, fid := range friendIDs {
		friendID := fid.(string)
		// 使用 Pipeline 批量添加通知
		pipe := s.redis.Pipeline()
		pipe.XAdd(ctx, &redis.XAddArgs{
			Stream: KEY_STATUS_NOTIFY,
			Values: map[string]interface{}{
				"user_id":    userID,
				"friend_id": friendID,
				"data":      string(notifyData),
			},
		})
		
		if _, err := pipe.Exec(ctx); err != nil {
			s.logger.Error("failed to add status notification",
				zap.Error(err),
				zap.String("friend_id", friendID))
		}
	}

	return nil
}

// GetUserInfo 获取用户信息
func (s *FriendService) GetUserInfo(ctx context.Context, userID int64) (*UserCenterInfo, error) {
	// 使用定义好的 KEY_USER_CENTER
	key := fmt.Sprintf(KEY_USER_CENTER, userID)
	data, err := s.redis.HGetAll(ctx, key).Result()
	if err != nil {
		return nil, fmt.Errorf("failed to get user info: %w", err)
	}

	info := &UserCenterInfo{}
	// 将 Redis hash 数据映射到结构体
	if err := mapstructure.Decode(data, info); err != nil {
		return nil, fmt.Errorf("failed to decode user info: %w", err)
	}

	return info, nil
}

// GetOnlineUsers 获取在线用户列表
func (s *FriendService) GetOnlineUsers(ctx context.Context) ([]int64, error) {
	// 使用定义好的 KEY_ONLINE_USERS
	result, err := s.redis.SMembers(ctx, KEY_ONLINE_USERS).Result()
	if err != nil {
		return nil, fmt.Errorf("failed to get online users: %w", err)
	}

	users := make([]int64, len(result))
	for i, id := range result {
		uid, err := strconv.ParseInt(id, 10, 64)
		if err != nil {
			s.logger.Error("invalid user id",
				zap.Error(err),
				zap.String("id", id))
			continue
		}
		users[i] = uid
	}

	return users, nil
}

// HandleFriendRequest 处理好友请求
func (s *FriendService) HandleFriendRequest(ctx context.Context, receiverID, senderID int64, accept bool) error {
	// 创建好友信息
	info := &FriendInfo{
		FriendID:   strconv.FormatInt(senderID, 10),
		CreateTime: time.Now().Unix(),
		Intimacy:   0,
	}

	// 序列化好友信息
	friendInfo, err := json.Marshal(info)
	if err != nil {
		return fmt.Errorf("failed to marshal friend info: %w", err)
	}

	// 设置请求状态
	status := REQUEST_STATUS_REJECTED
	if accept {
		status = REQUEST_STATUS_ACCEPTED
	}

	// 执行 Lua 脚本
	result, err := s.handleRequestScript.Run(ctx, s.redis,
		[]string{strconv.FormatInt(receiverID, 10), strconv.FormatInt(senderID, 10)},
		status,
		string(friendInfo),
	).Result()

	if err != nil {
		return fmt.Errorf("failed to handle friend request: %w", err)
	}

	// 处理结果
	res, ok := result.(map[string]interface{})
	if !ok {
		return errors.New("invalid script result")
	}

	if errMsg, exists := res["err"]; exists {
		return errors.New(errMsg.(string))
	}

	// 如果接受请求,发送通知
	if accept {
		// TODO: 发送添加好友成功的通知
		s.notifyFriendAdded(ctx, receiverID, senderID)
	}

	return nil
}

// CreateFriendRequest 创建好友请求
func (s *FriendService) CreateFriendRequest(ctx context.Context, senderID, receiverID int64, message string) error {
	req := &FriendRequest{
		SenderID:   strconv.FormatInt(senderID, 10),
		Message:    message,
		CreateTime: time.Now().Unix(),
	}

	// 序列化请求信息
	reqInfo, err := json.Marshal(req)
	if err != nil {
		return fmt.Errorf("failed to marshal request: %w", err)
	}

	key := fmt.Sprintf(KEY_FRIEND_REQ, receiverID)
	
	// 使用 Pipeline 执行操作
	pipe := s.redis.Pipeline()
	pipe.HSet(ctx, key, strconv.FormatInt(senderID, 10), string(reqInfo))
	pipe.Expire(ctx, key, time.Duration(REQUEST_EXPIRE_TIME)*time.Second)

	_, err = pipe.Exec(ctx)
	if err != nil {
		return fmt.Errorf("failed to create friend request: %w", err)
	}

	// TODO: 发送好友请求通知
	return nil
}

// UpdateIntimacy 更新好友度
func (s *FriendService) UpdateIntimacy(ctx context.Context, userID, friendID int64, action string) error {
	// 获取每日计数 key
	dailyKey := fmt.Sprintf(KEY_INTIMACY_DAILY, userID, friendID)
	
	// 检查是否超过每日上限
	count, err := s.redis.Get(ctx, dailyKey).Int()
	if err != nil && err != redis.Nil {
		return fmt.Errorf("failed to get daily intimacy count: %w", err)
	}
	
	if count >= INTIMACY_MAX_PER_DAY {
		return errors.New("reached daily intimacy limit")
	}
	
	// 确定增加的好友度
	var addValue int32
	switch action {
	case "chat":
		addValue = INTIMACY_ADD_CHAT
	case "game":
		addValue = INTIMACY_ADD_GAME
	case "gift":
		addValue = INTIMACY_ADD_GIFT
	default:
		return errors.New("invalid intimacy action")
	}
	
	// 使用 Pipeline 更新好友度
	pipe := s.redis.Pipeline()
	
	// 更新双方的好友信息
	for _, id := range []int64{userID, friendID} {
		key := fmt.Sprintf(KEY_FRIEND_HASH, id)
		targetID := friendID
		if id == friendID {
			targetID = userID
		}
		
		// 获取当前好友信息
		friendInfo := pipe.HGet(ctx, key, strconv.FormatInt(targetID, 10))
		
		// 更新好友度
		pipe.HIncrBy(ctx, key, fmt.Sprintf("%d:intimacy", targetID), int64(addValue))
	}
	
	// 更新每日计数
	pipe.Incr(ctx, dailyKey)
	pipe.Expire(ctx, dailyKey, time.Hour*24)
	
	// 执行 Pipeline
	if _, err := pipe.Exec(ctx); err != nil {
		return fmt.Errorf("failed to update intimacy: %w", err)
	}
	
	return nil
}

// StatusNotification 状态更新通知
type StatusNotification struct {
	UserID      int64  `json:"user_id"`
	Status      int32  `json:"status"`
	CustomStatus string `json:"custom_status"`
	CurrentGame string `json:"current_game"`
	UpdatedAt   int64  `json:"updated_at"`
}

// notifyFriendAdded 通知好友添加成功
func (s *FriendService) notifyFriendAdded(ctx context.Context, userID, friendID int64) {
	// 获取用户信息
	userInfo, err := s.GetUserInfo(ctx, userID)
	if err != nil {
		s.logger.Error("failed to get user info for notification",
			zap.Error(err),
			zap.Int64("user_id", userID))
		return
	}
	
	// 创建通知消息
	notify := map[string]interface{}{
		"type":      "friend_added",
		"user_id":   userID,
		"username":  userInfo.Username,
		"timestamp": time.Now().Unix(),
	}
	
	// 序列化通知
	notifyData, err := json.Marshal(notify)
	if err != nil {
		s.logger.Error("failed to marshal notification",
			zap.Error(err))
		return
	}
	
	// 添加到通知队列
	if err := s.redis.XAdd(ctx, &redis.XAddArgs{
		Stream: fmt.Sprintf("notify:user:%d", friendID),
		Values: map[string]interface{}{
			"type": "friend_added",
			"data": string(notifyData),
		},
	}).Err(); err != nil {
		s.logger.Error("failed to add notification",
			zap.Error(err),
			zap.Int64("friend_id", friendID))
	}
}
