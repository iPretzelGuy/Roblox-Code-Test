--[[ 
	Smart Sliding Door
	By: @iPretzelGuy

	Summary:
	- Tag any Model with CollectionService tag "SmartDoor"
	- The Model MUST have a PrimaryPart (this is the actual sliding panel)
	- Script auto-adds a ProximityPrompt to the Model
	- Door slides along local X or Z axis using CFrame math
	
	--[[
	This system implements easily configurable sliding doors using server-side CFrame
	animation rather than physics-based movement.

	The primary goals of this approach are:
	1) Deterministic behavior (doors always open/close exactly the same way)
	2) Low overhead (doors only update while animating)
	3) Simple configurability using Attributes instead of hardcoded values

	The system is fully data-driven:
	- Doors are discovered via CollectionService tags
	- Behavior is controlled through Model Attributes
	- Access is determined dynamically per-player

	!! This makes the system scalable and easy to maintain without modifying code
	when new doors are added.
]]

--// Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CollectionService = game:GetService("CollectionService")
local DataStoreService = game:GetService("DataStoreService")

--// DataStore (successful opens per player)
local DoorStatsStore = DataStoreService:GetDataStore("SmartDoor_OpenStats_v1")

--// Tag + prompt name
local DOOR_TAG = "SmartDoor"
local PROMPT_NAME = "SmartDoorPrompt"

--// Default settings (overridable via Attributes on the door Model)
local DEFAULTS = {
	ActionText = "Use Door",
	ObjectText = "Sliding Door",
	MaxActivationDistance = 10,
	HoldDuration = 0,

	-- Sliding settings
	SlideDistance = 10,     -- studs to slide when open
	SlideAxis = "X",        -- "X" or "Z" local axis

	-- Behavior
	OpenSpeed = 12,         -- larger means faster easing
	AutoCloseSeconds = 3,   -- 0 means never auto-close and stays open forever
	AllowToggle = true,     -- if false, only opens (auto-close handles closing)
	Locked = false,

	-- Access
	AccessLevel = 0,        -- 0 means that anyone can open the door
}

-- Attributes on the Door Model:
-- Locked (bool)
-- AccessLevel (number)
-- AutoCloseSeconds (number)
-- OpenSpeed (number)
-- AllowToggle (bool)
-- SlideDistance (number)
-- SlideAxis (string: "X"/"Z")
-- PromptActionText (string)
-- PromptObjectText (string)
-- Busy (bool)  -- used internally to debounce

-- Player clearance sources:
-- 1. player:GetAttribute("Clearance")
-- 2. Any equipped Tool:GetAttribute("Clearance") (keycard style)

--////////////////////////////////////////////////////////////
--// Small helpers
--////////////////////////////////////////////////////////////
local function clamp(n, a, b)
	if n < a then return a end
	if n > b then return b end
	return n
end

local function lerp(a, b, t)
	return a + (b - a) * t
end

local function expSmoothAlpha(speed, dt)
	-- stable smoothing even if dt varies
	return 1 - math.exp(-speed * dt)
end

local function safeNumber(v, fallback)
	local n = tonumber(v)
	if n == nil then return fallback end
	return n
end

local function safeBool(v, fallback)
	if v == nil then return fallback end
	return v == true
end

local function safeAxis(v)
	local s = tostring(v or DEFAULTS.SlideAxis):upper()
	if s ~= "X" and s ~= "Z" then
		return "X"
	end
	return s
end

--////////////////////////////////////////////////////////////
--// DoorController class
--////////////////////////////////////////////////////////////
-- DoorController represents one logical door instance.
-- Each tagged Model gets exactly one controller.
--
--  use a metatable-based class here to:
-- - Keep per-door state isolated
-- - Avoid global variables for door data
-- - Make cleanup predictable when doors are removed


local DoorController = {}
DoorController.__index = DoorController

function DoorController.new(doorModel: Model)
	assert(doorModel and doorModel:IsA("Model"), "SmartDoor must be a Model")
	assert(doorModel.PrimaryPart, "SmartDoor Model must have a PrimaryPart")

	local self = setmetatable({}, DoorController)

	self.Model = doorModel
	self.Part = doorModel.PrimaryPart -- the sliding panel

	self.ClosedCFrame = self.Part.CFrame

	-- Animation state
	self.IsOpen = false
	self.CurrentAlpha = 0
	self.TargetAlpha = 0
	self._movingConn = nil
	self._autoCloseTask = nil

	-- Setup
	self:ApplyDefaultsIfMissing()
	self:EnsurePrompt()
	self:ConnectCalibration()

	return self
end


-- ! ApplyDefaultsIfMissing:
-- Ensures all expected attributes exist on the Model.
-- This allows doors to work immediately without requiring
-- manual setup, while still letting designers override
-- any value directly in Studio.
function DoorController:ApplyDefaultsIfMissing()
	for key, defaultValue in pairs(DEFAULTS) do
		if self.Model:GetAttribute(key) == nil then
			self.Model:SetAttribute(key, defaultValue)
		end
	end
end

function DoorController:GetSetting(name)
	local v = self.Model:GetAttribute(name)
	if v == nil then
		return DEFAULTS[name]
	end
	return v
end

function DoorController:IsLocked()
	return safeBool(self:GetSetting("Locked"), false)
end

function DoorController:GetAccessLevel()
	return safeNumber(self:GetSetting("AccessLevel"), DEFAULTS.AccessLevel)
end

function DoorController:GetOpenSpeed()
	return math.max(0.1, safeNumber(self:GetSetting("OpenSpeed"), DEFAULTS.OpenSpeed))
end

function DoorController:GetAutoCloseSeconds()
	return math.max(0, safeNumber(self:GetSetting("AutoCloseSeconds"), DEFAULTS.AutoCloseSeconds))
end

function DoorController:AllowToggle()
	return safeBool(self:GetSetting("AllowToggle"), DEFAULTS.AllowToggle)
end

function DoorController:GetSlideDistance()
	return safeNumber(self:GetSetting("SlideDistance"), DEFAULTS.SlideDistance)
end

function DoorController:GetSlideAxis()
	return safeAxis(self:GetSetting("SlideAxis"))
end

function DoorController:EnsurePrompt()
	local prompt = self.Model:FindFirstChild(PROMPT_NAME)
	if not prompt then
		prompt = Instance.new("ProximityPrompt")
		prompt.Name = PROMPT_NAME
		prompt.Parent = self.Model
	end

	prompt.RequiresLineOfSight = false
	prompt.MaxActivationDistance = safeNumber(self:GetSetting("MaxActivationDistance"), DEFAULTS.MaxActivationDistance)
	prompt.HoldDuration = safeNumber(self:GetSetting("HoldDuration"), DEFAULTS.HoldDuration)

	prompt.ActionText = tostring(self.Model:GetAttribute("PromptActionText") or DEFAULTS.ActionText)
	prompt.ObjectText = tostring(self.Model:GetAttribute("PromptObjectText") or DEFAULTS.ObjectText)

	-- Ensure one connection
	if self._promptConn then
		if self._promptConn.Connected then self._promptConn:Disconnect() end
	end

	self._promptConn = prompt.Triggered:Connect(function(player)
		self:OnPrompt(player)
	end)
end

function DoorController:ConnectCalibration()
	-- Optional: flip this attribute to true in Studio to re-cache closed position
	self._calibConn = self.Model:GetAttributeChangedSignal("Recalibrate"):Connect(function()
		if self.Model:GetAttribute("Recalibrate") == true then
			self.ClosedCFrame = self.Part.CFrame
			self:SetOpen(false, true)
			self.Model:SetAttribute("Recalibrate", false)
		end
	end)
end

function DoorController:GetPlayerClearance(player: Player)
	local best = safeNumber(player:GetAttribute("Clearance"), 0)

	local char = player.Character
	if char then
		for _, inst in ipairs(char:GetChildren()) do
			if inst:IsA("Tool") then
				local lvl = safeNumber(inst:GetAttribute("Clearance"), 0)
				if lvl > best then
					best = lvl
				end
			end
		end
	end

	return best
end

function DoorController:CanUse(player: Player)
	if self:IsLocked() then
		return false, "Locked"
	end

	local required = self:GetAccessLevel()
	if required <= 0 then
		return true, "Public"
	end

	local clearance = self:GetPlayerClearance(player)
	if clearance >= required then
		return true, "Authorized"
	end

	return false, "NoAccess"
end

function DoorController:SetOpen(open: boolean, instant: boolean)
	self.IsOpen = open == true
	self.TargetAlpha = self.IsOpen and 1 or 0

	if instant then
		self.CurrentAlpha = self.TargetAlpha
		self.Part.CFrame = self:ComputeCFrame(self.CurrentAlpha)
		self:StopMotion()
	end
end

-- ! CinoyteCFrame
-- Computes the door’s position based on an alpha value (0–1).
-- Instead of storing absolute positions, we always offset
-- from the original closed CFrame. This prevents cumulative
-- floating-point drift and allows recalibration at runtime.
function DoorController:ComputeCFrame(alpha: number)
	-- Slide in local space relative to the CLOSED CFrame
	local dist = self:GetSlideDistance() * alpha
	local axis = self:GetSlideAxis()

	if axis == "X" then
		return self.ClosedCFrame * CFrame.new(dist, 0, 0)
	else
		return self.ClosedCFrame * CFrame.new(0, 0, dist)
	end
end

function DoorController:StopMotion()
	if self._movingConn then
		self._movingConn:Disconnect()
		self._movingConn = nil
	end
end

-- ! AnimateToTarger
-- Animates the door using exponential smoothing instead of
-- TweenService. This avoids creating extra objects and allows
-- smooth motion even if frame rate fluctuates.
--
-- The Heartbeat connection only exists while the door is moving,
-- which keeps idle performance effectively zero-cost.
function DoorController:AnimateToTarget()
	if self._movingConn then return end -- already moving

	local speed = self:GetOpenSpeed()

	self._movingConn = RunService.Heartbeat:Connect(function(dt)
		local t = expSmoothAlpha(speed, dt)
		self.CurrentAlpha = lerp(self.CurrentAlpha, self.TargetAlpha, t)

		-- Finish snap
		if math.abs(self.CurrentAlpha - self.TargetAlpha) < 0.002 then
			self.CurrentAlpha = self.TargetAlpha
			self.Part.CFrame = self:ComputeCFrame(self.CurrentAlpha)
			self:StopMotion()
			return
		end

		self.Part.CFrame = self:ComputeCFrame(self.CurrentAlpha)
	end)
end

function DoorController:QueueAutoClose()
	local seconds = self:GetAutoCloseSeconds()
	if seconds <= 0 then return end

	if self._autoCloseTask then
		task.cancel(self._autoCloseTask)
		self._autoCloseTask = nil
	end

	self._autoCloseTask = task.delay(seconds, function()
		self._autoCloseTask = nil
		if self.IsOpen then
			self:SetOpen(false, false)
			self:AnimateToTarget()
		end
	end)
end

function DoorController:RecordOpen(player: Player)
	-- Only write on a successful open
	task.spawn(function()
		local key = tostring(player.UserId)

		local okGet, current = pcall(function()
			return DoorStatsStore:GetAsync(key)
		end)
		if not okGet then return end

		local newValue = (safeNumber(current, 0) + 1)

		pcall(function()
			DoorStatsStore:SetAsync(key, newValue)
		end)
	end)
end

function DoorController:OnPrompt(player: Player)
	-- Small debounce so you can't spam the same door
	if self.Model:GetAttribute("Busy") == true then return end
	self.Model:SetAttribute("Busy", true)

	local allowed, reason = self:CanUse(player)
	if not allowed then
		self.Model:SetAttribute("LastDenied", reason)
		task.delay(0.2, function()
			if self.Model and self.Model.Parent then
				self.Model:SetAttribute("Busy", false)
			end
		end)
		return
	end

	if self.IsOpen then
		if self:AllowToggle() then
			self:SetOpen(false, false)
			self:AnimateToTarget()
		end
	else
		self:SetOpen(true, false)
		self:AnimateToTarget()
		self:QueueAutoClose()
		self:RecordOpen(player)
	end

	task.delay(0.15, function()
		if self.Model and self.Model.Parent then
			self.Model:SetAttribute("Busy", false)
		end
	end)
end

function DoorController:Destroy()
	self:StopMotion()

	if self._autoCloseTask then
		task.cancel(self._autoCloseTask)
		self._autoCloseTask = nil
	end

	if self._promptConn and self._promptConn.Connected then
		self._promptConn:Disconnect()
	end
	if self._calibConn and self._calibConn.Connected then
		self._calibConn:Disconnect()
	end
end

--////////////////////////////////////////////////////////////
--// System manager: create controllers for tagged doors
--////////////////////////////////////////////////////////////
local controllers = {} :: {[Instance]: any}

local function attachDoor(inst: Instance)
	if controllers[inst] then return end
	if not inst:IsA("Model") then return end
	if not inst.PrimaryPart then return end

	local ok, ctrl = pcall(function()
		return DoorController.new(inst)
	end)
	if ok and ctrl then
		controllers[inst] = ctrl

		-- Cleanup when removed
		inst.AncestryChanged:Connect(function(_, parent)
			if parent == nil then
				local c = controllers[inst]
				if c then
					controllers[inst] = nil
					c:Destroy()
				end
			end
		end)
	else
		warn("[SmartDoor] Failed to attach:", inst:GetFullName())
	end
end

-- Existing doors
for _, inst in ipairs(CollectionService:GetTagged(DOOR_TAG)) do
	attachDoor(inst)
end

-- New doors
CollectionService:GetInstanceAddedSignal(DOOR_TAG):Connect(function(inst)
	attachDoor(inst)
end)

CollectionService:GetInstanceRemovedSignal(DOOR_TAG):Connect(function(inst)
	local c = controllers[inst]
	if c then
		controllers[inst] = nil
		c:Destroy()
	end
end)

--////////////////////////////////////////////////////////////
--// Optional: demo clearance + leaderstats (for showing in-game)
--////////////////////////////////////////////////////////////
Players.PlayerAdded:Connect(function(player)
	-- Default to public clearance
	if player:GetAttribute("Clearance") == nil then
		player:SetAttribute("Clearance", 0)
	end

	-- Leaderstats (simple demo polish)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local doorsOpened = Instance.new("IntValue")
	doorsOpened.Name = "DoorsOpened"
	doorsOpened.Value = 0
	doorsOpened.Parent = leaderstats

	-- Load on join
	task.spawn(function()
		local ok, value = pcall(function()
			return DoorStatsStore:GetAsync(tostring(player.UserId))
		end)
		if ok and player.Parent then
			doorsOpened.Value = safeNumber(value, 0)
		end
	end)

	-- Low frequency refresh (not spamable)
	task.spawn(function()
		while player.Parent do
			task.wait(6)
			local ok, value = pcall(function()
				return DoorStatsStore:GetAsync(tostring(player.UserId))
			end)
			if ok and player.Parent then
				doorsOpened.Value = safeNumber(value, doorsOpened.Value)
			end
		end
	end)
end)
