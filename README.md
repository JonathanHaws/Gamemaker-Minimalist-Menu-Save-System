# Gamemaker-Minimalist-Menu-Save-System
A library I made for myself to prototype demos fast. No need for main menu. Pause menu with functioning save system, options and controls. Scalable and light. Make persisten player object and put these in. Also make ```audiogroup_effects``` and ```audiogroup_music```. Also special thanks the Freindly Cosmonaut for making json save functions.

## Game Start 
```
audio_group_load(audiogroup_music);
audio_group_load(audiogroup_effects);
load_options("options.txt");
load_game(global.save);
save_game(global.save);
```

## Create Event 
``` 
#region Menu

	function get_filename_for_new_save() {
		// Has to be unique or it will overwrite old saves
		return 
			string(current_year) + "." + 
			string(current_month) + "." + 
			string(current_day) + "_" + 
			string(current_hour) + "." + 
			string(current_minute) + "." + 
			string(current_second) + ".save"
		}
	
	function save_json(data, file_path) {
		var json_str = json_stringify(data); 
		var len = string_byte_length(json_str); 
		var buff = buffer_create(len, buffer_fixed, 1); 
		buffer_write(buff, buffer_text, json_str); 
		buffer_save(buff, file_path);
		buffer_delete(buff); 
		}	
	
	function load_json(file_path) { 
		var buff = buffer_load(file_path); 
		var str = buffer_read(buff, buffer_text); 
		var data = json_parse(str); 
		buffer_delete(buff); 
		return data; 
		}		
	
	function save_options(file_path) { 
		save_json({ 
			effects: audio_group_get_gain(audiogroup_effects),
			music: audio_group_get_gain(audiogroup_music),
			fullscreen: window_get_fullscreen(),
			last_played_save: global.save,
			}, file_path);
		}
	
	function load_options(file_path) {
		var data = file_exists(file_path) ? load_json(file_path) : {};
		
		if (!variable_struct_exists(data, "effects")) { data.effects = 1; } // Defaults
		if (!variable_struct_exists(data, "music")) { data.music = 1; }
		if (!variable_struct_exists(data, "fullscreen")) { data.fullscreen = 1; }
		if (!variable_struct_exists(data, "last_played_save")) { data.last_played_save = get_filename_for_new_save(); }
		
		audio_group_set_gain(audiogroup_effects, data.effects, 0); 
		audio_group_set_gain(audiogroup_music, data.music, 0); 
		window_set_fullscreen(data.fullscreen); 
		global.save = data.last_played_save;
		}
	
	function save_game(file_path) {
		save_json({ 
			x: x,
			y: y,
			room: room,
			}, file_path);
		}	
	
	function load_game(file_path) {
		var data = file_exists(file_path) ? load_json(file_path) : {};
		
		if (!variable_struct_exists(data, "x")) { data.x = x; }
		if (!variable_struct_exists(data, "y")) { data.y = y; }
		if (!variable_struct_exists(data, "room")) { data.room = 0; } // Defaults
		x = data.x;
		y = data.y; 
		if (room != data.room) { room_goto(data.room); }

		}
	
	function set_page(new_page = "main", new_selected = 0) {
		audio_play_sound(sound_menu, 1, false);
		selected = new_selected;
		menu = variable_instance_get(self.pages, new_page);
		}
	
	global.paused = false;
	
	pages = {
		main: [
			{ // Resume
				text: "Resume",
				pressed: function() { 
					global.paused = !global.paused; 
					audio_play_sound(sound_menu, 1, false); 
					} 
				}, 
			{ // Saves
				text: "Saves", 
				pressed: function() { other.set_page("saves"); } 
				},
			{ // Options
				text: "Options", 
				pressed: function() { other.set_page("options"); } 
				}, 
			{ // Controls
				text: "Controls", 
				pressed: function() { other.set_page("controls"); } 
				}, 
			{ // Exit
				text: "Exit", 
				pressed: function() { 
					other.save_game(global.save);
					game_end(); 
					} 
				}
			],
		saves: [
			{ // New Game
				text: "New Game",
				pressed: function () {
					global.save = other.get_filename_for_new_save(); 
					other.save_options("options.txt");
					game_restart()
					}
				},
			{ // Load Save
				text: "Load Save",
				pressed: function() {
					other.save_game(global.save);
					other.menu = [];
					var save_file = file_find_first("*.save", false);
					while (save_file != "") {	
					    if (save_file != global.save) {
							array_push(other.menu, { 
								text: save_file,
								pressed: function() { 
									global.save = self.text;
									other.load_game(global.save);
									global.paused = false;
									}
								});
							}
					    save_file = file_find_next();
						}
					array_push(other.menu, { text: "Back", pressed: function() { other.set_page("saves", 1)} });
					file_find_close();
					}
				}, 
			{ // Delete Save
				text: "Delete Save",
				pressed: function() {
					other.menu = [];
					var save_file = file_find_first("*.save", false);
					while (save_file != "") {
						if (save_file != global.save) {
						    array_push(other.menu, { 
								text: save_file,
								pressed: function() { 
									file_delete(self.text);
									array_delete(other.menu, self, 1);
									audio_play_sound(sound_menu, 1, false);
									}
								});
							}
					    save_file = file_find_next();
						}
					array_push(other.menu, { text: "Back", pressed: function() { other.set_page("saves", 2); } });
					file_find_close();
					}
				},
			{ // Back
				text: "Back", 
				pressed: function() { other.set_page("main", 1); } 
				},
			],
		options: [
			{ // Effects
				text: "Effects - " + string(round(audio_group_get_gain(audiogroup_effects) * 10)), 
				left_pressed: function (input = -.1) {
					var old_gain = audio_group_get_gain(audiogroup_effects);
					audio_group_set_gain(audiogroup_effects, clamp(old_gain + input, 0, 1) , 0);
					if (old_gain != audio_group_get_gain(audiogroup_effects)) { audio_play_sound(sound_menu, 1, false); }
					self.text = "Effects - " + string(round(audio_group_get_gain(audiogroup_effects) * 10));
					other.save_options("options.txt");
					},
				right_pressed: function() {
					self.left_pressed(.1); 
					}
				}, 
			{ // Music
				text: "Music - " + string(round(audio_group_get_gain(audiogroup_music) * 10)),
				left_pressed: function (input = -.1) {
					var old_gain = audio_group_get_gain(audiogroup_music);
					audio_group_set_gain(audiogroup_music, clamp(old_gain + input, 0, 1), 0);
					if (old_gain != audio_group_get_gain(audiogroup_music)) { audio_play_sound(sound_menu, 1, false); }
					self.text = "Music - " + string(round(audio_group_get_gain(audiogroup_music) * 10));
					other.save_options("options.txt");
					},
				right_pressed: function() {
					self.left_pressed(.1); 
					}
				},
			{ // Display
				text: "Display - " + (window_get_fullscreen() ? "Fullscreen" : "Windowed"),
				pressed: function () { 
					audio_play_sound(sound_menu, 1, false);
					window_set_fullscreen(!window_get_fullscreen()) 
					self.text = "Display - " + (window_get_fullscreen() ? "Fullscreen" : "Windowed");
					other.save_options("options.txt");
					} 
				},
			{ // Back
				text: "Back", 
				pressed: function() { other.set_page("main", 2); } 
				}
			],
		controls :[
			{ // Movment
				text: "WASD Space / Arrow Keys - Move" 
				},
			{ // Back
				text: "Back", 
				pressed: function() { other.set_page("main", 3); } 
				}
			], 	
		}		

	#endregion 
```
# Draw Gui End 
```
#region Navigate Pause Menu

	if (keyboard_check_pressed(vk_escape)) { // toggle
			global.paused = !global.paused 
			audio_play_sound(sound_menu, 1, false);
			set_page();
			} 
	
	if (!global.paused) { exit; }
	
	down_pressed = keyboard_check_pressed(ord("S")) || keyboard_check_pressed(vk_down);
	up_pressed = keyboard_check_pressed(ord("W")) || keyboard_check_pressed(vk_up);
	left_pressed = keyboard_check_pressed(ord("A")) || keyboard_check_pressed(vk_left);
	right_pressed = keyboard_check_pressed(ord("D")) || keyboard_check_pressed(vk_right);
	pressed = keyboard_check_pressed(vk_space) || keyboard_check_pressed(vk_enter);
	var lastSelected = selected; // Cycle
	selected = (selected + down_pressed - up_pressed + array_length(menu)) % array_length(menu);
	if (lastSelected != selected) { audio_play_sound(sound_menu, 1, false); }
	if (pressed) { if (variable_instance_exists(menu[selected], "pressed")) { menu[selected].pressed(); } }
	if (left_pressed) { if (variable_instance_exists(menu[selected], "left_pressed")) { menu[selected].left_pressed(); } }
	if (right_pressed) { if (variable_instance_exists(menu[selected], "left_pressed")) { menu[selected].right_pressed(); } }
	
	#endregion

#region Draw Pause Menu

	var centerX = camera_get_view_width(view_camera[0]) / 2;
	var centerY = camera_get_view_height(view_camera[0]) / 2;
	draw_set_color(c_black);
	draw_set_alpha(0.8);
	draw_rectangle(0, 0, display_get_width(), display_get_height(), false);
	draw_set_alpha(1);
	draw_set_halign(fa_center);
	draw_set_valign(fa_middle);	
	var menuLength = array_length(menu);
	var height = string_height("string to get height");
	for (var i = 0; i < menuLength; i++) {
		draw_set_color(c_dkgrey);
		if (selected == i) { draw_set_color(c_silver); } else { draw_set_color(c_grey); }
		draw_text(centerX, centerY - (menuLength / 2 * height) + (i * height), menu[i].text);
		}
		
	#endregion
 ```
