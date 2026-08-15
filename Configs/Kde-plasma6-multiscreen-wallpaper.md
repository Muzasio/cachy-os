topic: kde_plasma6_per_monitor_wallpaper
shell_note: fish_shell_needs_for_end_loop_and_single_quotes_for_dollar_signs

identify_monitors:
  cmd: xrandr --listmonitors
  gives: [connector_names, geometry_x_offset, geometry_y_offset]
  note: capture_both_x_and_y_offset_not_just_x_stacked_or_portrait_layouts_need_y_too

identify_monitor_model:
  cmd: |
    for f in /sys/class/drm/card*-<CONNECTOR>/edid
        strings "$f"
    end
  gives: monitor_model_name_per_connector

find_containment_ids:
  cmd: grep -B2 'Image=' ~/.config/plasma-org.kde.plasma.desktop-appletsrc
  gives: containment_id_to_current_wallpaper_path
  note: multiple_containment_ids_can_map_to_same_physical_screen_via_different_activities
  disambiguate_with: |
    grep -B1 'activityId=' ~/.config/plasma-org.kde.plasma.desktop-appletsrc
    # cross-check against: qdbus6 org.kde.plasmashell /org/kde/ActivityManager/Activities org.kde.ActivityManager.Activities.CurrentActivity
    # only the containment under the CURRENT active activity will visibly update

apply_wallpaper_per_screen:
  method: plasma_scripting_via_dbus
  note: screenName_property_unreliable_on_plasma6_use_geometry_offsets_instead
  binary_name_varies: try_qdbus6_then_qdbus_then_qdbus-qt6_depending_on_distro
  path_with_spaces_or_special_chars: url_encode_path_before_inserting_into_file_uri_or_use_a_symlink_without_spaces
  prereq: desktop_widgets_must_be_unlocked_else_writeConfig_fails_silently_with_DBus_Error_Failed
  cmd: |
    qdbus6 org.kde.plasmashell /PlasmaShell org.kde.PlasmaShell.evaluateScript '
    var d = desktops();
    for (i=0;i<d.length;i++) {
        var g = screenGeometry(d[i].screen);
        if (g.x == <TARGET_X_OFFSET> && g.y == <TARGET_Y_OFFSET>) {
            d[i].wallpaperPlugin = "org.kde.image";
            d[i].currentConfigGroup = Array("Wallpaper","org.kde.image","General");
            d[i].writeConfig("Image", "file://<ABSOLUTE_IMAGE_PATH>");
        }
    }
    '
  match_screen_by: x_and_y_offset_from_xrandr_listmonitors_output

fallback_single_screen_only:
  tool: plasma-apply-wallpaperimage
  cmd: plasma-apply-wallpaperimage <ABSOLUTE_IMAGE_PATH>
  limitation: applies_to_all_screens_or_current_screen_only_no_per_screen_targeting

verify_applied:
  cmd: grep -A2 'Containments\]\[<CONTAINMENT_ID>\]\[Wallpaper' ~/.config/plasma-org.kde.plasma.desktop-appletsrc

persistence:
  writes_to: ~/.config/plasma-org.kde.plasma.desktop-appletsrc
  survives: [reboot, logout, plasmashell_restart]
  breaks_if:
    - wallpaper_plugin_is_slideshow_not_org.kde.image
    - image_file_moved_or_deleted
