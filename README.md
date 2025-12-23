```typescript name=Header.tsx
import React, { useEffect, useRef, useState } from "react";
import { NavLink, Link, useLocation } from "react-router-dom";
import { Menu, X, Phone, Zap } from "lucide-react";
import { Button } from "@/components/ui/button";

const FOCUSABLE_SELECTORS =
  'a[href], area[href], input:not([disabled]):not([type="hidden"]), select:not([disabled]), textarea:not([disabled]), button:not([disabled]), [tabindex]:not([tabindex="-1"])';

const Header: React.FC = () => {
  const [isMenuOpen, setIsMenuOpen] = useState(false);
  const location = useLocation();
  const headerRef = useRef<HTMLElement | null>(null);
  const menuButtonRef = useRef<HTMLButtonElement | null>(null);
  const mobileNavRef = useRef<HTMLElement | null>(null);

  const navLinks = [
    { name: "Home", path: "/" },
    { name: "Services", path: "/services" },
    { name: "Reviews", path: "/reviews" },
    { name: "About", path: "/about" },
    { name: "Contact", path: "/contact" },
  ];

  // Close mobile menu when route changes
  useEffect(() => {
    setIsMenuOpen(false);
  }, [location.pathname]);

  // Manage focus trapping, Escape handling, body scroll lock, and inert/aria-hidden for the rest of the page
  useEffect(() => {
    const headerEl = headerRef.current;
    const mobileNavEl = mobileNavRef.current;
    if (!isMenuOpen) {
      // Cleanup: remove any attributes we possibly set on open
      // Restore body overflow
      document.body.style.overflow = "";
      // Remove inert / aria-hidden from body children
      Array.from(document.body.children).forEach((child) => {
        child.removeAttribute("aria-hidden");
        // @ts-ignore - inert is not in all TS lib DOM types
        if (child.hasAttribute("inert")) child.removeAttribute("inert");
      });
      return;
    }

    // Menu is open:
    // 1) Prevent background scroll
    const previousOverflow = document.body.style.overflow;
    document.body.style.overflow = "hidden";

    // 2) Set inert/aria-hidden on everything except this header (so screen readers won't navigate background)
    Array.from(document.body.children).forEach((child) => {
      if (child === headerEl) return;
      child.setAttribute("aria-hidden", "true");
      // @ts-ignore - inert isn't standard in lib.dom.d.ts everywhere, but browsers support it increasingly; set anyway
      try {
        child.setAttribute("inert", "true");
      } catch {
        // ignore
      }
    });

    // 3) Focus management: focus first focusable inside mobileNav
    const focusable = mobileNavEl
      ? Array.from(mobileNavEl.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS)).filter(
          (el) => el.offsetParent !== null || el.getAttribute("tabindex") !== null
        )
      : [];

    if (focusable.length > 0) {
      focusable[0].focus();
    } else {
      // fallback: focus close button
      menuButtonRef.current?.focus();
    }

    // 4) Key handling: Escape closes and returns focus to menu button; Tab is trapped
    const onKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Escape") {
        setIsMenuOpen(false);
        menuButtonRef.current?.focus();
      }

      if (e.key === "Tab" && mobileNavEl) {
        const elems = Array.from(mobileNavEl.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS)).filter(
          (el) => el.offsetParent !== null || el.getAttribute("tabindex") !== null
        );
        if (elems.length === 0) {
          e.preventDefault();
          return;
        }
        const first = elems[0];
        const last = elems[elems.length - 1];
        if (e.shiftKey && document.activeElement === first) {
          // shift+Tab on first -> move to last
          e.preventDefault();
          last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
          // Tab on last -> move to first
          e.preventDefault();
          first.focus();
        }
      }
    };

    window.addEventListener("keydown", onKeyDown);

    return () => {
      window.removeEventListener("keydown", onKeyDown);
      document.body.style.overflow = previousOverflow;
      Array.from(document.body.children).forEach((child) => {
        child.removeAttribute("aria-hidden");
        // @ts-ignore
        if (child.hasAttribute("inert")) child.removeAttribute("inert");
      });
    };
  }, [isMenuOpen]);

  // Click outside mobile nav to close (on overlay)
  const handleOverlayClick = (e: React.MouseEvent) => {
    // close when clicking the overlay area
    setIsMenuOpen(false);
    menuButtonRef.current?.focus();
  };

  return (
    <header
      ref={headerRef}
      className="sticky top-0 z-50 bg-background/95 backdrop-blur-sm border-b border-border"
    >
      <div className="container mx-auto px-4">
        <div className="flex items-center justify-between h-16 md:h-20">
          {/* Logo */}
          <NavLink to="/" className="flex items-center gap-2 group" aria-label="N.G. Electrical homepage">
            <div className="flex items-center justify-center w-10 h-10 rounded-lg gradient-electric">
              <Zap className="w-5 h-5 text-primary-foreground" />
            </div>
            <span className="font-display text-xl font-bold text-foreground group-hover:text-primary transition-colors">
              N.G. Electrical
            </span>
          </NavLink>

          {/* Desktop Navigation */}
          <nav className="hidden md:flex items-center gap-1" role="navigation" aria-label="Main navigation">
            {navLinks.map((link) => (
              <NavLink
                key={link.path}
                to={link.path}
                className={({ isActive }) =>
                  `px-4 py-2 rounded-lg text-sm font-medium transition-colors ${
                    isActive ? "bg-primary text-primary-foreground" : "text-muted-foreground hover:text-foreground hover:bg-secondary"
                  }`
                }
                aria-current={(location.pathname === link.path && "page") || undefined}
              >
                {link.name}
              </NavLink>
            ))}
          </nav>

          {/* CTA Button & Mobile Menu */}
          <div className="flex items-center gap-3">
            <Button asChild className="hidden sm:inline-flex gradient-electric hover:opacity-90 transition-opacity">
              <Link to="/contact">
                <Phone className="w-4 h-4 mr-2" />
                Get a Quote
              </Link>
            </Button>

            {/* Mobile Menu Button */}
            <button
              ref={menuButtonRef}
              onClick={() => setIsMenuOpen((s) => !s)}
              className="md:hidden p-2 rounded-lg hover:bg-secondary transition-colors"
              aria-label={isMenuOpen ? "Close menu" : "Open menu"}
              aria-expanded={isMenuOpen}
              aria-controls="mobile-navigation"
            >
              {isMenuOpen ? <X className="w-6 h-6" /> : <Menu className="w-6 h-6" />}
            </button>
          </div>
        </div>

        {/* Mobile Navigation + overlay */}
        {isMenuOpen && (
          <>
            {/* Overlay to catch outside clicks; placed under the nav but above page content */}
            <div
              onClick={handleOverlayClick}
              aria-hidden="true"
              className="fixed inset-0 z-40 bg-black/40 backdrop-blur-sm md:hidden"
            />

            <nav
              id="mobile-navigation"
              ref={mobileNavRef}
              className="md:hidden py-4 border-t border-border motion-safe:animate-fade-in relative z-50"
              role="dialog"
              aria-label="Mobile navigation"
              aria-modal="true"
            >
              <div className="flex flex-col gap-2 px-4">
                {navLinks.map((link) => (
                  <NavLink
                    key={link.path}
                    to={link.path}
                    onClick={() => {
                      setIsMenuOpen(false);
                      // return focus to menu button after close (small timeout for navigation)
                      setTimeout(() => menuButtonRef.current?.focus(), 0);
                    }}
                    className={({ isActive }) =>
                      `block px-4 py-3 rounded-lg font-medium transition-colors ${
                        isActive ? "bg-primary text-primary-foreground" : "text-muted-foreground hover:text-foreground hover:bg-secondary"
                      }`
                    }
                    aria-current={(location.pathname === link.path && "page") || undefined}
                  >
                    {link.name}
                  </NavLink>
                ))}

                <Button asChild className="mt-2 gradient-electric">
                  <Link
                    to="/contact"
                    onClick={() => {
                      setIsMenuOpen(false);
                      setTimeout(() => menuButtonRef.current?.focus(), 0);
                    }}
                  >
                    <Phone className="w-4 h-4 mr-2" />
                    Get a Quote
                  </Link>
                </Button>
              </div>
            </nav>
          </>
        )}
      </div>
    </header>
  );
};

export default Header;
```
