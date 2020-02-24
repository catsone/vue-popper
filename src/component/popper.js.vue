<style>
.popper {
  width: auto;
  background-color: #fafafa;
  color: #212121;
  text-align: center;
  padding: 2px;
  display: inline-block;
  border-radius: 3px;
  position: absolute;
  font-size: 14px;
  font-weight: normal;
  border: 1px #ebebeb solid;
  z-index: 200000;
  -moz-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  -webkit-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  box-shadow: rgb(58, 58, 58) 0 0 6px 0;
}

.popper .popper__arrow {
  width: 0;
  height: 0;
  border-style: solid;
  position: absolute;
  /* margin: 5px; */
}

.popper[data-popper-placement^="top"] {
  margin-bottom: 5px;
}

.popper[data-popper-placement^="top"] .popper__arrow {
  border-width: 5px 5px 0 5px;
  border-color: #fafafa transparent transparent transparent;
  bottom: -5px;
  left: calc(50% - 5px);
  margin-top: 0;
  margin-bottom: 0;
}

/* .popper[data-popper-placement^="bottom"] {
      margin-top: 5px;
    } */

.popper[data-popper-placement^="bottom"] .popper__arrow {
  border-width: 0 5px 5px 5px;
  border-color: transparent transparent #fafafa transparent;
  top: -5px;
  left: calc(50% - 5px);
  margin-top: 0;
  margin-bottom: 0;
}

/* .popper[data-popper-placement^="right"] {
      margin-left: 5px;
    } */

.popper[data-popper-placement^="right"] .popper__arrow {
  border-width: 5px 5px 5px 0;
  border-color: transparent #fafafa transparent transparent;
  left: -5px;
  top: calc(50% - 5px);
  margin-left: 0;
  margin-right: 0;
}

/* .popper[data-popper-placement^="left"] {
      margin-right: 5px;
    } */

.popper[data-popper-placement^="left"] .popper__arrow {
  border-width: 5px 0 5px 5px;
  border-color: transparent transparent transparent #fafafa;
  right: -5px;
  top: calc(50% - 5px);
  margin-left: 0;
  margin-right: 0;
}
</style>

<template>
  <component :is="tagName">
    <transition
      :name="transition"
      :enter-active-class="enterActiveClass"
      :leave-active-class="leaveActiveClass"
      @after-leave="doDestroy"
    >
      <span ref="popper" :class="rootClass" v-show="!disabled && showPopper">
        <slot>{{ content }}</slot>
      </span>
    </transition>
    <slot name="reference"></slot>
  </component>
</template>

<script>
import { createPopper } from "@popperjs/core";

function on(element, event, handler) {
  if (element && event && handler) {
    document.addEventListener
      ? element.addEventListener(event, handler, false)
      : element.attachEvent("on" + event, handler);
  }
}

function off(element, event, handler) {
  if (element && event) {
    document.removeEventListener
      ? element.removeEventListener(event, handler, false)
      : element.detachEvent("on" + event, handler);
  }
}

export default {
  props: {
    tagName: {
      type: String,
      default: "span"
    },
    trigger: {
      type: String,
      default: "hover",
      validator: value =>
        [
          "clickToOpen",
          "click", // Same as clickToToggle, provided for backwards compatibility.
          "clickToToggle",
          "hover",
          "focus",
          "programmatic"
        ].indexOf(value) > -1
    },
    delayOnMouseOver: {
      type: Number,
      default: 10
    },
    delayOnMouseOut: {
      type: Number,
      default: 10
    },
    disabled: {
      type: Boolean,
      default: false
    },
    content: String,
    enterActiveClass: String,
    leaveActiveClass: String,
    boundariesSelector: String,
    reference: {},
    forceShow: {
      type: Boolean,
      default: false
    },
    dataValue: {
      default: null
    },
    appendToBody: {
      type: Boolean,
      default: false
    },
    visibleArrow: {
      type: Boolean,
      default: true
    },
    transition: {
      type: String,
      default: ""
    },
    stopPropagation: {
      type: Boolean,
      default: false
    },
    preventDefault: {
      type: Boolean,
      default: false
    },
    options: {
      type: Object,
      default() {
        return {};
      }
    },
    rootClass: {
      type: String,
      default: ""
    }
  },

  data() {
    return {
      referenceElm: null,
      popperJS: null,
      showPopper: false,
      currentPlacement: "",
      popperOptions: {
        placement: "bottom",
        modifiers: [
          {
            name: "computeStyle",
            options: {
              gpuAcceleration: false
            }
          }
        ]
      }
    };
  },

  watch: {
    showPopper(value) {
      if (value) {
        this.$emit("show", this);
        if (this.popperJS) {
          const eventListenersIndex = this.popperJS.state.options.modifiers.findIndex(
            modifier => modifier.name === "eventListeners"
          );

          if (eventListenersIndex > -1) {
            this.popperJS.state.options.modifiers[
              eventListenersIndex
            ].options = {
              scroll: true,
              resize: true
            };
          } else {
            this.popperJS.state.options.modifiers.push({
              name: "eventListeners",
              options: {
                scroll: true,
                resize: true
              }
            });
          }
        }
        this.updatePopper();
      } else {
        if (this.popperJS) {
          const eventListenersIndex = this.popperJS.state.options.modifiers.findIndex(
            modifier => modifier.name === "eventListeners"
          );

          if (eventListenersIndex > -1) {
            this.popperJS.state.options.modifiers[
              eventListenersIndex
            ].options = {
              scroll: false,
              resize: false
            };
          } else {
            this.popperJS.state.options.modifiers.push({
              name: "eventListeners",
              options: {
                scroll: false,
                resize: false
              }
            });
          }
        }
        this.$emit("hide", this);
      }
    },

    forceShow: {
      handler(value) {
        this[value ? "doShow" : "doClose"]();
      },
      immediate: true
    },

    disabled(value) {
      if (value) {
        this.showPopper = false;
      }
    }
  },

  created() {
    this.appendedArrow = false;
    this.appendedToBody = false;
    this.popperOptions = { ...this.popperOptions, ...this.options };
  },

  mounted() {
    this.referenceElm = this.reference || this.$slots.reference[0].elm;
    this.popper = this.$slots.default[0].elm;

    switch (this.trigger) {
      case "clickToOpen":
        on(this.referenceElm, "click", this.doShow);
        on(document, "click", this.handleDocumentClick);
        break;
      case "click": // Same as clickToToggle, provided for backwards compatibility.
      case "clickToToggle":
        on(this.referenceElm, "click", this.doToggle);
        on(document, "click", this.handleDocumentClick);
        break;
      case "hover":
        on(this.referenceElm, "mouseover", this.onMouseOver);
        on(this.popper, "mouseover", this.onMouseOver);
        on(this.referenceElm, "mouseout", this.onMouseOut);
        on(this.popper, "mouseout", this.onMouseOut);
        break;
      case "focus":
        on(this.referenceElm, "focus", this.onMouseOver);
        on(this.popper, "focus", this.onMouseOver);
        on(this.referenceElm, "blur", this.onMouseOut);
        on(this.popper, "blur", this.onMouseOut);
        break;
      case "programmatic":
        break;
    }
  },

  updated() {
    this.popperOptions = { ...this.popperOptions, ...this.options };
    this.$nextTick(this.updatePopper);
  },

  methods: {
    doToggle(event) {
      if (this.stopPropagation) {
        event.stopPropagation();
      }

      if (this.preventDefault) {
        event.preventDefault();
      }

      if (!this.forceShow) {
        this.showPopper = !this.showPopper;
      }
    },

    doShow() {
      this.showPopper = true;
    },

    doClose() {
      this.showPopper = false;
    },

    doDestroy() {
      if (this.showPopper) {
        return;
      }

      if (this.popperJS) {
        this.popperJS.destroy();
        this.popperJS = null;
      }

      if (this.appendedToBody) {
        this.appendedToBody = false;
        document.body.removeChild(this.popper.parentElement);
      }
    },

    makePopper() {
      this.$nextTick(() => {
        if (this.visibleArrow) {
          this.appendArrow(this.popper);
        }

        if (this.appendToBody && !this.appendedToBody) {
          this.appendedToBody = true;
          document.body.appendChild(this.popper.parentElement);
        }

        if (this.popperJS && this.popperJS.destroy) {
          this.popperJS.destroy();
        }

        if (this.boundariesSelector) {
          const boundariesElement = document.querySelector(
            this.boundariesSelector
          );

          if (boundariesElement) {
            this.popperOptions.modifiers = [...this.popperOptions.modifiers];

            const preventOverflowModifierIndex = this.popperOptions.modifiers.findIndex(
              modifier => modifier.name === "preventOverflow"
            );

            if (preventOverflowModifierIndex) {
              this.popper.modifiers[preventOverflowModifierIndex] = {
                name: "preventOverflow",
                options: {
                  ...this.popper.modifiers[preventOverflowModifierIndex]
                    .options,
                  boundary: boundariesElement
                }
              };
            }
          }
        }

        this.popperOptions.onFirstUpdate = () => {
          this.$emit("created", this);
          this.$nextTick(this.updatePopper);
        };

        const reference = this.anchorElm ? this.anchorElm : this.referenceElm;

        this.popperJS = createPopper(
          reference,
          this.popper,
          this.popperOptions
        );
      });
    },

    destroyPopper() {
      off(this.referenceElm, "click", this.doToggle);
      off(this.referenceElm, "mouseup", this.doClose);
      off(this.referenceElm, "mousedown", this.doShow);
      off(this.referenceElm, "focus", this.doShow);
      off(this.referenceElm, "blur", this.doClose);
      off(this.referenceElm, "mouseout", this.onMouseOut);
      off(this.referenceElm, "mouseover", this.onMouseOver);
      off(document, "click", this.handleDocumentClick);

      this.showPopper = false;
      this.doDestroy();
    },

    appendArrow(element) {
      if (this.appendedArrow) {
        return;
      }

      this.appendedArrow = true;

      const arrow = document.createElement("div");
      arrow.setAttribute("data-popper-arrow", "");
      arrow.className = "popper__arrow";
      element.appendChild(arrow);
    },

    updatePopper() {
      this.popperJS ? this.popperJS.update() : this.makePopper();
    },

    onMouseOver() {
      clearTimeout(this._timer);
      this._timer = setTimeout(() => {
        this.showPopper = true;
      }, this.delayOnMouseOver);
    },

    onMouseOut() {
      clearTimeout(this._timer);
      this._timer = setTimeout(() => {
        this.showPopper = false;
      }, this.delayOnMouseOut);
    },

    handleDocumentClick(e) {
      if (
        !this.$el ||
        !this.referenceElm ||
        this.elementContains(this.$el, e.target) ||
        this.elementContains(this.referenceElm, e.target) ||
        !this.popper ||
        this.elementContains(this.popper, e.target)
      ) {
        return;
      }

      this.$emit("documentClick", this);

      if (this.forceShow) {
        return;
      }

      this.showPopper = false;
    },

    elementContains(elm, otherElm) {
      if (typeof elm.contains === "function") {
        return elm.contains(otherElm);
      }

      return false;
    }
  },

  destroyed() {
    this.destroyPopper();
  }
};
</script>
