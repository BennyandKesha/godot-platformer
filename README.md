extends Area

var taken = false

func _on_coin_body_enter(body):
	if not taken and body is preload("res://player/player.gd"):
		get_node("Animation").play("take")
		taken = true
    extends RigidBody

const ACCEL = 5.0
const DEACCEL = 20.0
const MAX_SPEED = 2.0
const ROT_SPEED = 1.0

var prev_advance = false
var dying = false
var rot_dir = 4

onready var gravity = ProjectSettings.get_setting("physics/3d/default_gravity") * ProjectSettings.get_setting("physics/3d/default_gravity_vector")


func _integrate_forces(state):
	var delta = state.get_step()
	var lv = state.get_linear_velocity()
	var g = state.get_total_gravity()
	# get_total_gravity returns zero for the first few frames, leading to errors.
	if g == Vector3.ZERO:
		g = gravity

	lv += g * delta # Apply gravity.
	var up = -g.normalized()

	if dying:
		state.set_linear_velocity(lv)
		return

	for i in range(state.get_contact_count()):
		var cc = state.get_contact_collider_object(i)
		var dp = state.get_contact_local_normal(i)

		if cc:
			if cc is preload("res://player/bullet/bullet.gd") and cc.enabled:
				set_mode(MODE_RIGID)
				dying = true
				state.set_angular_velocity(-dp.cross(up).normalized() * 33.0)
				get_node("AnimationPlayer").play("impact")
				get_node("AnimationPlayer").queue("explode")
				cc.enabled = false
				get_node("SoundHit").play()
				return

	var col_floor = get_node("Armature/RayFloor").is_colliding()
	var col_wall = get_node("Armature/RayWall").is_colliding()

	var advance = col_floor and not col_wall

	var dir = get_node("Armature").get_transform().basis[2].normalized()
	var deaccel_dir = dir

	if advance:
		if dir.dot(lv) < MAX_SPEED:
			lv += dir * ACCEL * delta
		deaccel_dir = dir.cross(g).normalized()
	else:
		if prev_advance:
			rot_dir = 1

		dir = Basis(up, rot_dir * ROT_SPEED * delta).xform(dir)
		get_node("Armature").set_transform(Transform().looking_at(-dir, up))

	var dspeed = deaccel_dir.dot(lv)
	dspeed -= DEACCEL * delta
	if dspeed < 0:
		dspeed = 0

	lv = lv - deaccel_dir * deaccel_dir.dot(lv) + deaccel_dir * dspeed

	state.set_linear_velocity(lv)
	prev_advance = advance


func _die():
	queue_free()
extends KinematicBody

enum Anim {
	FLOOR,
	AIR,
}

const SHOOT_TIME = 1.5
const SHOOT_SCALE = 2
const CHAR_SCALE = Vector3(0.3, 0.3, 0.3)
const MAX_SPEED = 4.5
const TURN_SPEED = 40
const JUMP_VELOCITY = 8.5
const BULLET_SPEED = 20
const AIR_IDLE_DEACCEL = false
const ACCEL = 14.0
const DEACCEL = 14.0
const AIR_ACCEL_FACTOR = 0.4
const SHARP_TURN_THRESHOLD = 140

var movement_dir = Vector3()
var linear_velocity = Vector3()
var jumping = false
var prev_shoot = false
var shoot_blend = 0

onready var gravity = ProjectSettings.get_setting("physics/3d/default_gravity") * ProjectSettings.get_setting("physics/3d/default_gravity_vector")


func _ready():
	get_node("AnimationTree").set_active(true)


func _physics_process(delta):
	linear_velocity += gravity * delta

	var anim = Anim.FLOOR

	var vv = linear_velocity.y # Vertical velocity.
	var hv = Vector3(linear_velocity.x, 0, linear_velocity.z) # Horizontal velocity.

	var hdir = hv.normalized() # Horizontal direction.
	var hspeed = hv.length() # Horizontal speed.

	# Player input.
	var cam_basis = get_node("Target/Camera").get_global_transform().basis
	var dir = Vector3() # Where does the player intend to walk to.
	dir = (Input.get_action_strength("move_right") - Input.get_action_strength("move_left")) * cam_basis[0]
	dir += (Input.get_action_strength("move_backwards") - Input.get_action_strength("move_forward")) * cam_basis[2]
	dir.y = 0
	dir = dir.normalized()

	var jump_attempt = Input.is_action_pressed("jump")
	var shoot_attempt = Input.is_action_pressed("shoot")

	if is_on_floor():
		var sharp_turn = hspeed > 0.1 and rad2deg(acos(dir.dot(hdir))) > SHARP_TURN_THRESHOLD

		if dir.length() > 0.1 and !sharp_turn:
			if hspeed > 0.001:
				hdir = adjust_facing(hdir, dir, delta, 1.0 / hspeed * TURN_SPEED, Vector3.UP)
			else:
				hdir = dir

			if hspeed < MAX_SPEED:
				hspeed += ACCEL * delta
		else:
			hspeed -= DEACCEL * delta
			if hspeed < 0:
				hspeed = 0

		hv = hdir * hspeed

		var mesh_xform = get_node("Armature").get_transform()
		var facing_mesh = -mesh_xform.basis[0].normalized()
		facing_mesh = (facing_mesh - Vector3.UP * facing_mesh.dot(Vector3.UP)).normalized()

		if hspeed > 0:
			facing_mesh = adjust_facing(facing_mesh, dir, delta, 1.0 / hspeed * TURN_SPEED, Vector3.UP)
		var m3 = Basis(-facing_mesh, Vector3.UP, -facing_mesh.cross(Vector3.UP).normalized()).scaled(CHAR_SCALE)

		get_node("Armature").set_transform(Transform(m3, mesh_xform.origin))

		if not jumping and jump_attempt:
			vv = JUMP_VELOCITY
			jumping = true
			get_node("SoundJump").play()
	else:
		anim = Anim.AIR

		if dir.length() > 0.1:
			hv += dir * (ACCEL * AIR_ACCEL_FACTOR * delta)
			if hv.length() > MAX_SPEED:
				hv = hv.normalized() * MAX_SPEED
		elif AIR_IDLE_DEACCEL:
			hspeed = hspeed - (DEACCEL * AIR_ACCEL_FACTOR * delta)
			if hspeed < 0:
				hspeed = 0
			hv = hdir * hspeed

	if jumping and vv < 0:
		jumping = false

	linear_velocity = hv + Vector3.UP * vv

	if is_on_floor():
		movement_dir = linear_velocity

	linear_velocity = move_and_slide(linear_velocity, -gravity.normalized())

	if shoot_blend > 0:
		shoot_blend -= delta * SHOOT_SCALE
		if (shoot_blend < 0):
			shoot_blend = 0

	if shoot_attempt and not prev_shoot:
		shoot_blend = SHOOT_TIME
		var bullet = preload("res://player/bullet/bullet.tscn").instance()
		bullet.set_transform(get_node("Armature/Bullet").get_global_transform().orthonormalized())
		get_parent().add_child(bullet)
		bullet.set_linear_velocity(get_node("Armature/Bullet").get_global_transform().basis[2].normalized() * BULLET_SPEED)
		bullet.add_collision_exception_with(self) # Add it to bullet.
		get_node("SoundShoot").play()

	prev_shoot = shoot_attempt

	if is_on_floor():
		$AnimationTree["parameters/walk/blend_amount"] = hspeed / MAX_SPEED

	$AnimationTree["parameters/state/current"] = anim
	$AnimationTree["parameters/air_dir/blend_amount"] = clamp(-linear_velocity.y / 4 + 0.5, 0, 1)
	$AnimationTree["parameters/gun/blend_amount"] = min(shoot_blend, 1.0)


func adjust_facing(p_facing, p_target, p_step, p_adjust_rate, current_gn):
	var n = p_target # Normal.
	var t = n.cross(current_gn).normalized()

	var x = n.dot(p_facing)
	var y = t.dot(p_facing)

	var ang = atan2(y,x)

	if abs(ang) < 0.001: # Too small.
		return p_facing

	var s = sign(ang)
	ang = ang * s
	var turn = ang * p_adjust_rate * p_step
	var a
	if ang < turn:
		a = ang
	else:
		a = turn
	ang = (ang - a) * s

	return (n * cos(ang) + t * sin(ang)) * p_facing.l
