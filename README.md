- name: Modify Layout
  env:
    DECODE_DIR: decoded
  run: |
    python3 <<'PY'
    import os
    import re
    from pathlib import Path

    ROOT = Path(os.environ["DECODE_DIR"])
    LAYOUT_DIR = ROOT / "res" / "layout"

    # =========================================================
    # القيمة التي يقرأها التطبيق للسيرفر.
    #
    # ضع هنا فقط عنوان السيرفر الذي تملكه/مصرح لك باستخدامه.
    # إذا كان الـAPK يحتوي أصلاً على القيمة الصحيحة، اتركها None
    # حتى يحافظ السكربت على القيمة الموجودة.
    # =========================================================

    DEFAULT_SERVER_URL = None

    ACTIVATION_HINT = "أدخل رمز التفعيل"


    # ---------------------------------------------------------
    # Helpers
    # ---------------------------------------------------------

    def add_or_replace_attribute(tag, attr, value):
        """
        إضافة android:attribute أو استبدالها داخل XML tag.
        """

        pattern = re.compile(
            rf'\s+android:{re.escape(attr)}\s*=\s*["\'][^"\']*["\']',
            re.IGNORECASE
        )

        replacement = f' android:{attr}="{value}"'

        if pattern.search(tag):
            return pattern.sub(
                replacement,
                tag,
                count=1
            )

        return tag[:-1].rstrip() + replacement + ">"


    def is_input_tag(tag):
        return bool(
            re.search(
                r'<\s*(EditText|AutoCompleteTextView|'
                r'com\.[^ >]*EditText|'
                r'androidx\.[^ >]*EditText)',
                tag,
                re.IGNORECASE
            )
        )


    def get_attr(tag, name):
        match = re.search(
            rf'android:{re.escape(name)}\s*=\s*["\']([^"\']*)["\']',
            tag,
            re.IGNORECASE
        )

        return match.group(1) if match else ""


    def looks_like_server(tag):
        text = tag.lower()

        keywords = (
            "server",
            "serverurl",
            "server_url",
            "url",
            "host",
            "baseurl",
            "base_url",
            "link"
        )

        # id / hint / text / input-related attributes
        for keyword in keywords:
            if keyword in text:
                return True

        return False


    def looks_like_password(tag):
        text = tag.lower()

        keywords = (
            "password",
            "passwd",
            "pwd",
            "pass"
        )

        return any(
            keyword in text
            for keyword in keywords
        )


    # =========================================================
    # Search layouts
    # =========================================================

    if not LAYOUT_DIR.exists():
        raise SystemExit(
            "❌ res/layout directory not found."
        )


    modified_files = 0
    server_fields = 0
    password_fields = 0
    activation_fields = 0


    for xml_file in LAYOUT_DIR.rglob("*.xml"):

        try:
            text = xml_file.read_text(
                encoding="utf-8"
            )
        except Exception:
            continue


        # -----------------------------------------------------
        # Find XML tags containing EditText
        # -----------------------------------------------------

        pattern = re.compile(
            r'<[^<>]*(?:EditText|AutoCompleteTextView)[^<>]*>',
            re.IGNORECASE
        )

        changed = False


        def process_tag(match):

            nonlocal changed
            nonlocal server_fields
            nonlocal password_fields
            nonlocal activation_fields

            tag = match.group(0)

            if not is_input_tag(tag):
                return tag


            # =================================================
            # SERVER FIELD
            # =================================================

            if looks_like_server(tag):

                server_fields += 1

                # Hide visually
                tag = add_or_replace_attribute(
                    tag,
                    "visibility",
                    "gone"
                )

                # -------------------------------------------------
                # IMPORTANT:
                #
                # If the original XML already contains android:text,
                # preserve it.
                #
                # If DEFAULT_SERVER_URL is explicitly supplied,
                # use that value.
                # -------------------------------------------------

                existing_text = get_attr(
                    tag,
                    "text"
                )

                if DEFAULT_SERVER_URL:
                    tag = add_or_replace_attribute(
                        tag,
                        "text",
                        DEFAULT_SERVER_URL
                    )

                    print(
                        f"  SERVER: {xml_file} -> "
                        f"using configured server URL"
                    )

                elif existing_text:
                    print(
                        f"  SERVER: {xml_file} -> "
                        f"preserving existing value"
                    )

                else:
                    print(
                        f"  SERVER: {xml_file} -> "
                        f"hidden, but no server value injected"
                    )

                changed = True

                return tag


            # =================================================
            # PASSWORD FIELD
            # =================================================

            if looks_like_password(tag):

                password_fields += 1

                # Hide visually.
                #
                # We deliberately preserve the APK's existing
                # value instead of injecting credentials.
                tag = add_or_replace_attribute(
                    tag,
                    "visibility",
                    "gone"
                )

                changed = True

                print(
                    f"  PASSWORD: {xml_file} -> hidden"
                )

                return tag


            return tag


        new_text = pattern.sub(
            process_tag,
            text
        )


        if new_text != text:

            xml_file.write_text(
                new_text,
                encoding="utf-8"
            )

            modified_files += 1


    # =========================================================
    # Create a simple visible activation field
    # =========================================================

    activation_layout = (
        LAYOUT_DIR /
        "dhiqar_activation.xml"
    )

    activation_layout.write_text(
        """<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp">

    <EditText
        android:id="@+id/dhiqar_activation_code"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="أدخل رمز التفعيل"
        android:text=""
        android:singleLine="true"
        android:inputType="text" />

</LinearLayout>
""",
        encoding="utf-8"
    )


    # =========================================================
    # Summary
    # =========================================================

    print("")
    print("========================================")
    print("LAYOUT MODIFICATION COMPLETE")
    print("========================================")
    print(f"Modified XML files : {modified_files}")
    print(f"Server fields      : {server_fields}")
    print(f"Password fields    : {password_fields}")
    print(f"Activation layout   : {activation_layout}")
    print("========================================")
    PY