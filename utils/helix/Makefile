# Helix edit by Urlynn

include $(TOPDIR)/rules.mk

PKG_NAME:=helix
PKG_VERSION:=25.08
PKG_RELEASE:=1
PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/helix-editor/helix.git
PKG_SOURCE_VERSION:=HEAD
PKG_MIRROR_HASH:=skip
PKG_BUILD_DIR:=$(BUILD_DIR)/helix-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

# --- TARGET ------------------------------------------------------------------
RUST_TARGET:=x86_64-unknown-linux-musl
RUST_TARGET_ENV:=X86_64_UNKNOWN_LINUX_MUSL
CPU_TARGET:=alderlake
# -----------------------------------------------------------------------------

define Package/helix
  SECTION:=utils
  CATEGORY:=Utilities
  SUBMENU:=Editors
  TITLE:=A post-modern modal text editor
  URL:=https://helix-editor.com
endef

define Package/helix/description
  A Kakoune / Neovim inspired editor, written in Rust.
endef

define Package/helix-runtime
  DEPENDS:=+libstdcpp
  CATEGORY:=Utilities
  SUBMENU:=Editors
  TITLE:=Helix runtime data (grammars, themes, queries)
  URL:=https://helix-editor.com
endef

define Build/Prepare
	$(call Build/Prepare/Default)
	sed -i 's/except = \[.*\]/only = ["bash","c","cpp","comment","css","diff","fish","hosts","html","ini","javascript","json","lua","make","markdown","nginx","python","regex","ssh_client_config","toml","xml","yaml"]/' $(PKG_BUILD_DIR)/languages.toml
endef

define Build/Configure
endef

define Build/Compile
	cd $(PKG_BUILD_DIR) && \
	HELIX_DEFAULT_RUNTIME="/usr/lib/helix/runtime" \
	CARGO_$(RUST_TARGET_ENV)_LINKER="$(TARGET_CROSS)gcc" \
	RUSTUP_TOOLCHAIN=stable \
	RUSTFLAGS="-C target-feature=-crt-static -C target-cpu=$(CPU_TARGET) -C lto=yes -C codegen-units=1 -C panic=abort" \
	cargo build --release --target $(RUST_TARGET) --locked
endef

define Package/helix/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/target/$(RUST_TARGET)/release/hx $(1)/usr/bin/
	$(TARGET_CROSS)strip $(1)/usr/bin/hx
endef

define Package/helix-runtime/install
	$(INSTALL_DIR) $(1)/usr/lib/helix/runtime
	$(CP) $(PKG_BUILD_DIR)/runtime/* $(1)/usr/lib/helix/runtime/
	rm -rf $(1)/usr/lib/helix/runtime/grammars/sources
endef

$(eval $(call BuildPackage,helix))
$(eval $(call BuildPackage,helix-runtime))
