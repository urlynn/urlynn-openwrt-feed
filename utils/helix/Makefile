include $(TOPDIR)/rules.mk

PKG_NAME:=helix
PKG_VERSION:=25.07.1
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define Package/helix
  SECTION:=utils
  CATEGORY:=Utilities
  SUBMENU:=Editors
  TITLE:=A post-modern modal text editor
  URL:=https://helix-editor.com
end Package/helix

define Build/Prepare
  git clone https://github.com/helix-editor/helix
  cd helix
endef

# Reproducible
cargo install --path helix-term --locked
# Optimized

cargo install \
   --profile opt \
   --config 'build.rustflags="-C target-cpu="alderlake' \
   --path helix-term \
   --locked

define Build/Compile
	RUSTFLAGS="-C target-feature=-crt-static" \
  --target x86_64-unknown-linux-musl \
	$(Build/Compile/Cargo) --release
endef

define Package/pslinkb/install
	# Install
endef

$(eval $(call BuildPackage,helix))
