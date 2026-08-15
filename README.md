-- Gojo Infinity - Auto-Run (No GUI)
-- Execute and it runs automatically

local Config = {
	ScanRadius = 200,
	BoxWidth = 10,
	BoxHeight = 10,
	BoxDepth = 10,
	BoxOffsetY = 0,
	MaxStep = 4,
	UrgentStep = 9,
	PredictTime = 0.12,
	LeadTime = 0.35,
	UrgentTime = 0.16,
	ExtraMargin = 0.35,
	ClearMargin = 0.6,
	MaxSubsteps = 20,
	SubstepStud = 2,
	Predictive = true,
	Vertical = true,
	KeepVelocity = true,
	NoDrag = true,
	CenterDeadzone = 0.6,
	SideCommitTime = 0.4,
	Smoothing = 0.55,
	UrgentSmoothing = 1,
	MinStep = 0.03,
	VerticalDamp = 0.35,
	SlowSpeed = 2,
	ClearanceProbe = 14,
	ClearanceMin = 5,
	WallPadding = 1.2,
	GroundDrop = 60,
	RefreshInterval = 0.2,
	SphereMinSpeed = 4,
	SphereMinSize = 3,
	MaxBalls = 24,
	BallsFolderName = "Balls",
	PracticePath = { "Lobby", "PracticeArea", "PracticeBalls" },
	BallKeywords = { "ball", "practice" },
}

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer

local v3new = Vector3.new
local v3zero = Vector3.zero
local mHuge = math.huge
local mMin = math.min
local mMax = math.max
local mAbs = math.abs
local mCeil = math.ceil
local mClamp = math.clamp
local tInsert = table.insert
local sLower = string.lower
local sFind = string.find

-- Cleanup previous
if getgenv().GojoInfinity then
	pcall(function() getgenv().GojoInfinity:Disconnect() end)
	getgenv().GojoInfinity = nil
end

local BallCache = {}
local BallVelocity = {}
local BallSample = {}
local BlockedAxes = {}
local BallsContainer, PracticeContainer
local lastRefresh = 0
local sideSign = 1
local sideChosenAt = 0
local lastEngaged = 0

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude
rayParams.IgnoreWater = true

local Character, Humanoid, Root

local function updateFilter()
	local filter = {}
	if Character then tInsert(filter, Character) end
	if BallsContainer then tInsert(filter, BallsContainer) end
	if PracticeContainer then tInsert(filter, PracticeContainer) end
	rayParams.FilterDescendantsInstances = filter
end

local function bindCharacter(char)
	Character = char
	Humanoid = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
	Root = char:FindFirstChild("HumanoidRootPart") or char:WaitForChild("HumanoidRootPart", 5)
	updateFilter()
end

if LocalPlayer.Character then
	bindCharacter(LocalPlayer.Character)
end

LocalPlayer.CharacterAdded:Connect(bindCharacter)

local function resolvePath(root, path)
	local node = root
	for _, name in ipairs(path) do
		if not node then return nil end
		node = node:FindFirstChild(name)
	end
	return node
end

local function matchesKeyword(name)
	local lowered = sLower(name)
	for _, keyword in ipairs(Config.BallKeywords) do
		if sFind(lowered, keyword, 1, true) then return true end
	end
	return false
end

local function isPlayerModel(instance)
	local model = instance:FindFirstAncestorOfClass("Model")
	while model do
		if Players:GetPlayerFromCharacter(model) then return true end
		model = model:FindFirstAncestorOfClass("Model")
	end
	return false
end

local function isSphereBall(part)
	if not part:IsA("Part") then return false end
	if part.Shape ~= Enum.PartType.Ball then return false end
	if part.Size.X < Config.SphereMinSize then return false end
	if part.AssemblyLinearVelocity.Magnitude < Config.SphereMinSpeed then return false end
	return not isPlayerModel(part)
end

local function collectFrom(container, parts, seen)
	for _, descendant in ipairs(container:GetDescendants()) do
		if descendant:IsA("BasePart") and not seen[descendant] then
			seen[descendant] = true
			tInsert(parts, descendant)
		end
	end
end

local function refreshBalls(now)
	if now - lastRefresh < Config.RefreshInterval then return end
	lastRefresh = now
	local rescan = false
	if not BallsContainer or not BallsContainer.Parent then
		BallsContainer = Workspace:FindFirstChild(Config.BallsFolderName)
		rescan = true
	end
	if not PracticeContainer or not PracticeContainer.Parent then
		PracticeContainer = resolvePath(Workspace, Config.PracticePath)
		rescan = true
	end
	if rescan then updateFilter() end
	local parts = {}
	local seen = {}
	if BallsContainer then collectFrom(BallsContainer, parts, seen) end
	if PracticeContainer then collectFrom(PracticeContainer, parts, seen) end
	for _, child in ipairs(Workspace:GetChildren()) do
		if child ~= BallsContainer and not Players:GetPlayerFromCharacter(child) then
			if child:IsA("BasePart") then
				if not seen[child] and (matchesKeyword(child.Name) or isSphereBall(child)) then
					seen[child] = true
					tInsert(parts, child)
				end
			elseif child:IsA("Folder") or child:IsA("Model") then
				if matchesKeyword(child.Name) then
					collectFrom(child, parts, seen)
				else
					for _, sub in ipairs(child:GetChildren()) do
						if sub:IsA("BasePart") and not seen[sub] then
							if matchesKeyword(sub.Name) or isSphereBall(sub) then
								seen[sub] = true
								tInsert(parts, sub)
							end
						end
					end
				end
			end
		end
	end
	for i = #BallCache, 1, -1 do BallCache[i] = nil end
	for index, part in ipairs(parts) do
		if index > Config.MaxBalls then break end
		tInsert(BallCache, part)
	end
	for part in pairs(BallSample) do
		if not part.Parent then
			BallSample[part] = nil
			BallVelocity[part] = nil
		end
	end
end

local function velocityOf(part, position, now)
	local velocity = part.AssemblyLinearVelocity
	if velocity.Magnitude > 1 then
		BallVelocity[part] = velocity
		BallSample[part] = { position, now }
		return velocity
	end
	local sample = BallSample[part]
	if sample then
		local delta = now - sample[2]
		if delta > 0.05 then
			local estimated = (position - sample[1]) / delta
			BallVelocity[part] = estimated
			BallSample[part] = { position, now }
			return estimated
		end
		return BallVelocity[part] or velocity
	end
	BallSample[part] = { position, now }
	return BallVelocity[part] or velocity
end

local function boxGap(localPoint, radius)
	local dx = mAbs(localPoint.X) - (Config.BoxWidth * 0.5 + radius)
	local dy = mAbs(localPoint.Y - Config.BoxOffsetY) - (Config.BoxHeight * 0.5 + radius)
	local dz = mAbs(localPoint.Z) - (Config.BoxDepth * 0.5 + radius)
	return mMax(dx, mMax(dy, dz))
end

local function clearanceAlong(origin, direction)
	local probe = Workspace:Raycast(origin, direction * Config.ClearanceProbe, rayParams)
	if not probe then return Config.ClearanceProbe end
	return probe.Distance
end

local function chooseSide(rootCFrame, lateralWorld, offsetAlongLateral, now)
	local preferred
	if mAbs(offsetAlongLateral) > Config.CenterDeadzone then
		preferred = offsetAlongLateral >= 0 and -1 or 1
	else
		preferred = sideSign
	end
	if now - sideChosenAt < Config.SideCommitTime then return sideSign end
	local origin = rootCFrame.Position
	local room = clearanceAlong(origin, lateralWorld * preferred)
	if room < Config.ClearanceMin then
		local otherRoom = clearanceAlong(origin, lateralWorld * -preferred)
		if otherRoom > room then preferred = -preferred end
	end
	if preferred ~= sideSign then
		sideSign = preferred
		sideChosenAt = now
	end
	return sideSign
end

local function boxExtentAlong(axis)
	return Config.BoxWidth * 0.5 * mAbs(axis.X)
		+ Config.BoxHeight * 0.5 * mAbs(axis.Y)
		+ Config.BoxDepth * 0.5 * mAbs(axis.Z)
end

local function staticPushout(localPoint, radius)
	local halfX = Config.BoxWidth * 0.5 + radius + Config.ExtraMargin
	local halfY = Config.BoxHeight * 0.5 + radius + Config.ExtraMargin
	local halfZ = Config.BoxDepth * 0.5 + radius + Config.ExtraMargin
	local offsetY = localPoint.Y - Config.BoxOffsetY
	local px = halfX - mAbs(localPoint.X)
	local py = halfY - mAbs(offsetY)
	local pz = halfZ - mAbs(localPoint.Z)
	if px <= 0 or py <= 0 or pz <= 0 then return nil end
	local axis = "X"
	local depth = px
	if pz < depth then axis = "Z"; depth = pz end
	if Config.Vertical and py * (1 / Config.VerticalDamp) < depth then
		axis = "Y"; depth = py
	end
	if axis == "X" then
		return localPoint.X >= 0 and v3new(-1, 0, 0) or v3new(1, 0, 0), depth
	elseif axis == "Z" then
		return localPoint.Z >= 0 and v3new(0, 0, -1) or v3new(0, 0, 1), depth
	end
	return offsetY >= 0 and v3new(0, -1, 0) or v3new(0, 1, 0), depth
end

local function safeOffset(origin, direction, distance)
	local result = Workspace:Raycast(origin, direction * (distance + Config.WallPadding), rayParams)
	if not result then return distance end
	local allowed = result.Distance - Config.WallPadding
	if allowed <= 0.05 then return 0 end
	return mMin(distance, allowed)
end

local function stripDrag(push)
	if not Config.NoDrag then return push end
	for index = 1, #BlockedAxes do
		local axis = BlockedAxes[index]
		local along = push:Dot(axis)
		if along > 0 then push = push - axis * along end
	end
	return push
end

local function applyPush(push, urgent, now)
	push = stripDrag(push)
	local length = push.Magnitude
	if length < Config.MinStep then return false end
	local direction = push.Unit
	if urgent then
		length = mMin(length * Config.UrgentSmoothing, Config.UrgentStep)
	else
		length = mMin(length * Config.Smoothing, Config.MaxStep)
	end
	if length < Config.MinStep then return false end
	local origin = Root.Position
	local allowed = safeOffset(origin, direction, length)
	if allowed <= 0.02 then
		local side = v3new(-direction.Z, 0, direction.X)
		if side.Magnitude < 0.05 then return false end
		side = stripDrag(side.Unit)
		if side.Magnitude < 0.05 then return false end
		side = side.Unit
		local sideAllowed = safeOffset(origin, side, length)
		if sideAllowed <= 0.02 then
			local flipped = stripDrag(-side)
			if flipped.Magnitude < 0.05 then return false end
			side = flipped.Unit
			sideAllowed = safeOffset(origin, side, length)
		end
		if sideAllowed <= 0.02 then return false end
		sideSign = -sideSign
		sideChosenAt = now
		direction = side
		allowed = sideAllowed
	end
	local offset = direction * allowed
	if mAbs(offset.Y) < 0.05 then
		local ground = Workspace:Raycast(Root.Position + offset + v3new(0, 5, 0), v3new(0, -Config.GroundDrop, 0), rayParams)
		if not ground then return false end
	end
	local velocity = Root.AssemblyLinearVelocity
	Root.CFrame = Root.CFrame + offset
	if Config.KeepVelocity then
		Root.AssemblyLinearVelocity = velocity
	end
	return true
end

local function resolveOnce(now, deltaTime)
	if not Root or not Root.Parent or not Humanoid or Humanoid.Health <= 0 then
		return false
	end
	local rootCFrame = Root.CFrame
	local origin = rootCFrame.Position
	local ownVelocity = Root.AssemblyLinearVelocity
	local sweepHorizon = deltaTime + (Config.Predictive and Config.PredictTime or 0)
	local leadHorizon = sweepHorizon + Config.LeadTime
	local push = v3zero
	local urgent = false
	local count = 0
	for i = #BlockedAxes, 1, -1 do BlockedAxes[i] = nil end
	for _, part in ipairs(BallCache) do
		if part and part.Parent then
			local position = part.Position
			if (position - origin).Magnitude < Config.ScanRadius then
				local radius = part.Size.X * 0.5
				local velocity = velocityOf(part, position, now)
				local relative = velocity - ownVelocity
				local speed = relative.Magnitude
				local travel = relative * leadHorizon
				local steps = 1
				local span = travel.Magnitude
				if span > Config.SubstepStud then
					steps = mClamp(mCeil(span / Config.SubstepStud), 1, Config.MaxSubsteps)
				end
				local threatened = false
				local threatTime = mHuge
				for stepIndex = 0, steps do
					local fraction = stepIndex / steps
					local samplePoint = position + travel * fraction
					local localPoint = rootCFrame:PointToObjectSpace(samplePoint)
					if boxGap(localPoint, radius) < Config.ExtraMargin then
						threatened = true
						threatTime = fraction * leadHorizon
						break
					end
				end
				if threatened then
					local escape = nil
					if speed > Config.SlowSpeed then
						local approach = relative.Unit
						tInsert(BlockedAxes, approach)
						local lateral = v3new(approach.Z, 0, -approach.X)
						if lateral.Magnitude > 0.05 then
							lateral = lateral.Unit
							local offsetVector = position - origin
							local separation = offsetVector:Dot(lateral)
							local sign = chooseSide(rootCFrame, lateral, separation, now)
							local localLateral = rootCFrame:VectorToObjectSpace(lateral)
							local corridor = boxExtentAlong(localLateral) + radius + Config.ExtraMargin + Config.ClearMargin
							local needed = corridor - mAbs(separation)
							if separation * sign > 0 then
								needed = corridor + mAbs(separation)
							end
							if needed > 0 then
								escape = lateral * sign * needed
							end
						end
					end
					if not escape then
						local localPoint = rootCFrame:PointToObjectSpace(position)
						local direction, depth = staticPushout(localPoint, radius)
						if direction then
							escape = rootCFrame:VectorToWorldSpace(direction) * depth
						end
					end
					if escape then
						if threatTime <= Config.UrgentTime then urgent = true end
						push = push + escape
						count = count + 1
					end
				end
			end
		end
	end
	if count == 0 then return false end
	lastEngaged = now
	push = v3new(push.X, push.Y * Config.VerticalDamp, push.Z)
	return applyPush(push, urgent, now)
end

local function gojoStep(deltaTime)
	local now = tick()
	refreshBalls(now)
	if not Root or not Root.Parent or not Humanoid or Humanoid.Health <= 0 then
		return
	end
	if now - lastEngaged > 1 then sideChosenAt = 0 end
	for _ = 1, 3 do
		local ok, result = pcall(resolveOnce, now, deltaTime)
		if not ok or not result then break end
	end
end

-- Start immediately
getgenv().GojoInfinity = RunService.Heartbeat:Connect(function(dt)
	local ok, err = pcall(gojoStep, dt)
	if not ok then warn("Gojo Infinity error: " .. tostring(err)) end
end)

-- Unload function
getgenv().ScriptUnload = function()
	if getgenv().GojoInfinity then
		pcall(function() getgenv().GojoInfinity:Disconnect() end)
		getgenv().GojoInfinity = nil
	end
	getgenv().ScriptUnload = nil
end

print("Gojo Infinity - Auto Running")
