# FCJ Internship Report Website

An **Internship Report Site** built with [Hugo](https://gohugo.io/) and the [Hugo Learn Theme](https://github.com/matcornic/hugo-theme-learn), designed to document work logs, proposals, technical blogs, workshop projects, and evaluations during the **First Cloud AI Journey (FCJ)** Bootcamp at **Amazon Web Services (AWS) Vietnam**.

---

## 📌 Project Overview

This repository serves as a structured online report portfolio for the AWS Vietnam Workforce Bootcamp. It features bilingual support (English & Vietnamese) and markdown-driven content organization.

### 👤 Student Details
- **Full Name:** Nguyen Tien Tho
- **University:** Sai Gon University
- **Major:** Information Technology (Class: DCT1224)
- **Company:** Amazon Web Services Vietnam Company Limited
- **Program:** Workforce Bootcamp - First Cloud AI Journey
- **Duration:** June 22, 2026 – August 15, 2026

---

## ✨ Features

- **Hugo Static Site Generator:** Fast build times and lightweight static output.
- **Hugo Learn Theme:** Clean, documentation-style navigation with collapsible sidebar chapters and built-in search.
- **Multilingual Support:** Supports both English (`.md`) and Vietnamese (`.vi.md`) content.
- **Structured Content Categories:**
  1. **Worklog:** Weekly & daily activity tracking.
  2. **Proposal:** Project proposals and technical plans.
  3. **Blogs Posted:** Technical articles and published content.
  4. **Events Participated:** AWS community events and workshops attended.
  5. **Workshop:** Hands-on lab tutorials and architecture projects.
  6. **Self-Evaluation:** Growth reflection and performance assessment.
  7. **Sharing & Feedback:** Program feedback and mentorship notes.
- **Vercel / Cloud Deployment Ready:** Pre-configured for deployment with custom Hugo versions.

---

## 📁 Project Structure

```text
.
├── archetypes/          # Content templates for new pages
├── content/             # Site content written in Markdown (.md and .vi.md)
│   ├── 1-Worklog/
│   ├── 2-Proposal/
│   ├── 3-BlogsPosted/
│   ├── 4-EventParticipated/
│   ├── 5-Workshop/
│   ├── 6-Self-evaluation/
│   ├── 7-Feedback/
│   ├── _index.md        # Home page (English)
│   └── _index.vi.md     # Home page (Vietnamese)
├── layouts/             # Custom HTML layout overrides
├── static/              # Static assets (images, CSS, JS)
│   └── images/          # Profile pictures and illustration assets
├── themes/              # Hugo themes (Submodule: hugo-theme-learn)
├── config.toml          # Main Hugo configuration file
├── vercel.json          # Vercel build configuration
└── README.md            # Project documentation
```

---

## 🛠️ Prerequisites

Before running the project locally, ensure you have the following installed:

1. **Git**: [Download Git](https://git-scm.com/)
2. **Hugo (Extended version recommended)**: [Download Hugo](https://gohugo.io/installation/)
   - Verify installation:
     ```bash
     hugo version
     ```
   - *Note: Hugo version `0.134.3` or higher is recommended.*

---

## 🚀 Getting Started / How to Run

### 1. Clone the Repository

Since the project uses Git submodules for the theme, clone with the `--recursive` flag:

```bash
git clone --recursive https://github.com/tientho201/fcj-workshop-template.git
cd fcj-workshop-template
```

If you have already cloned the repository without `--recursive`, initialize and update submodules manually:

```bash
git submodule update --init --recursive
```

### 2. Run Locally (Development Server)

Start the local Hugo server with draft mode enabled:

```bash
hugo server -D
```

- Open your browser and navigate to: `http://localhost:1313/`
- The site will automatically reload when changes are made to content or configuration files.

### 3. Build for Production

To build the static HTML files for production:

```bash
hugo --minify
```

The generated static website files will be output to the `public/` directory.

---

## 🌐 Deployment

### Deploying to Vercel

1. Push your code to a GitHub/GitLab repository.
2. Import the project into [Vercel](https://vercel.com).
3. Vercel will automatically detect `vercel.json` and build the project using Hugo.

### Deploying to GitHub Pages

You can also use GitHub Actions to deploy to GitHub Pages automatically upon pushing to the `main` branch.

---

## 📝 License & Acknowledgments

- **Theme:** [Hugo Learn Theme](https://github.com/matcornic/hugo-theme-learn)
- **Organization:** AWS Study Group / First Cloud AI Journey (FCJ)
