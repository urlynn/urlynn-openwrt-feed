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

define Build/Prepare
	$(call Build/Prepare/Default)
endef

define Build/Configure
endef

define Build/Compile
	cd $(PKG_BUILD_DIR) && \
	HELIX_DEFAULT_RUNTIME="/usr/lib/helix/runtime" \
	CARGO_$(RUST_TARGET_ENV)_LINKER="$(TARGET_CROSS)gcc" \
	RUSTFLAGS="-C target-feature=-crt-static -C target-cpu=$(CPU_TARGET)" \
	cargo build --release --target $(RUST_TARGET) --locked
endef

define Package/helix/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_DIR) $(1)/usr/lib/helix/runtime
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/target/$(RUST_TARGET)/release/hx $(1)/usr/bin/
	$(CP) $(PKG_BUILD_DIR)/runtime/* $(1)/usr/lib/helix/runtime/
endef

$(eval $(call BuildPackage,helix))
