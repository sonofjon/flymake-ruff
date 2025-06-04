# flymake-ruff

[![MELPA](https://melpa.org/packages/flymake-ruff-badge.svg)](https://melpa.org/#/flymake-ruff)

Flymake plugin to run a linter for python buffers using [ruff](https://github.com/charliermarsh/ruff)

> Make sure you have at least `ruff` `0.1.0` version because there is a [breaking change](https://github.com/astral-sh/ruff/blob/main/BREAKING_CHANGES.md#010) in the output format flag
> If you are using a version of `ruff` < `0.5.0`, set flymake-ruff-program-args to  `'("--output-format" "text" "--exit-zero" "--quiet" "-")`

## Installation

### Cloning the repo

Clone this repo somewhere, and add this to your config:

```elisp
(add-to-list 'load-path "path where the repo was cloned")

(require 'flymake-ruff)
(add-hook 'python-mode-hook #'flymake-ruff-load)
```

### Using use-package

```emacs-lisp
(use-package flymake-ruff
  :ensure t
  :hook (python-mode . flymake-ruff-load))
```

### Using straight.el

```emacs-lisp
(use-package flymake-ruff
  :straight (flymake-ruff
             :type git
             :host github
             :repo "erickgnavar/flymake-ruff"))
```

## Using flymake-ruff with eglot

To use flymake-ruff together with eglot, you should add `flymake-ruff-load` to
`eglot-managed-mode-hook` instead.   For example:

```emacs-lisp
(add-hook 'eglot-managed-mode-hook 'flymake-ruff-load)
```

Or, if you use use-package:

```emacs-lisp
(use-package flymake-ruff
  :ensure t
  :hook (eglot-managed-mode . flymake-ruff-load))
```

## Excluding Pyright diagnostic notes

Pyright has some diagnostic notes that overlap with diagnostics provided by
ruff. These diagnostic notes can't be disabled via Pyright's config, but you can
exclude them by adding a filter to `eglot--report-to-flymake`. For example, to
remove Pyright's "variable not accessed" notes, add the following:

```emacs-lisp
(defun my-filter-eglot-diagnostics (diags)
    "Drop Pyright 'variable not accessed' notes from DIAGS."
    (list (seq-remove (lambda (d)
                        (and (eq (flymake-diagnostic-type d) 'eglot-note)
                             (s-starts-with? "Pyright:" (flymake-diagnostic-text d))
                             (s-ends-with? "is not accessed" (flymake-diagnostic-text d))))
                      (car diags))))

(advice-add 'eglot--report-to-flymake :filter-args #'my-filter-eglot-diagnostics))
```

## Jump to Ruff rule documentation

The following helper scans the Flymake diagnostics buffer at point for a
“[RULE123]”-style code and opens its reference page at
https://docs.astral.sh/ruff/rules.

```elisp
;; Open Ruff docs from Flymake
(defun flymake-ruff-goto-doc ()
"Browse to the documentation for the Ruff rule on a Flymake diagnostic line.

Scans the Flymake diagnostic at point for a “[RULE123]”-style code and
browses to its documentation at https://docs.astral.sh/ruff/rules."
(interactive)
(unless (or (derived-mode-p 'flymake-diagnostics-buffer-mode)
            (derived-mode-p 'flymake-project-diagnostics-mode))
    (user-error "Not in a Flymake diagnostics buffer"))
  (let* ((id (tabulated-list-get-id))
         (diag (or (plist-get id :diagnostic)
                   (user-error "Bad Flymake ID: %S" id)))
         (msg (flymake-diagnostic-text diag)))
    (unless (string-match (rx "[" (group (1+ upper-case) (1+ digit)) "]")
                          msg)
      (user-error "No Ruff rule (like [RULE123]) in diagnostic: %s" msg))
    (browse-url
     (format "https://docs.astral.sh/ruff/rules/%s"
             (match-string 1 msg)))))

(with-eval-after-load 'flymake
  (define-key flymake-diagnostics-buffer-mode-map
              (kbd "M-RET") #'flymake-ruff-goto-doc)
  (define-key flymake-project-diagnostics-mode-map
              (kbd "M-RET") #'flymake-ruff-goto-doc))
```

Now, when you’re in any Flymake diagnostics buffer, pressing `M-RET` on a line containing a Ruff rule will open the corresponding rule page in your browser.
