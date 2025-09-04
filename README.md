# test_piezoelectric_report.py
#140450140003002031,nader.maleki
import unittest
import os
import tempfile
import shutil
import json
import pandas as pd
import numpy as np
from unittest.mock import patch, MagicMock
import matplotlib.pyplot as plt

# Import the functions from the main code (after refactoring)
from main_code import (
    create_title_page,
    process_seed_data,
    simulate_piezoelectric_sensor,
    create_innovation_page,
    generate_report
)

class TestPiezoelectricReport(unittest.TestCase):

    def setUp(self):
        """Set up test environment"""
        self.test_dir = tempfile.mkdtemp()
        self.results_dir = os.path.join(self.test_dir, 'results')
        os.makedirs(self.results_dir, exist_ok=True)

        # Create test data files
        self.create_test_files()

    def tearDown(self):
        """Clean up test environment"""
        shutil.rmtree(self.test_dir)
        plt.close('all')

    def create_test_files(self):
        """Create test JSON, CSV files for testing"""
        # Create metrics JSON files
        for i in range(2):
            metrics_data = {
                'test_accuracy': 0.85 + i*0.05,
                'f1_score': 0.82 + i*0.04
            }
            with open(os.path.join(self.results_dir, f'metrics_seed{i}.json'), 'w') as f:
                json.dump(metrics_data, f)

        # Create history CSV files
        for i in range(2):
            history_data = {
                'epoch': range(10),
                'accuracy': [0.1*j for j in range(10)],
                'val_accuracy': [0.08*j for j in range(10)]
            }
            df = pd.DataFrame(history_data)
            df.to_csv(os.path.join(self.results_dir, f'history_seed{i}.csv'), index=False)

        # Create confusion matrix CSV files
        for i in range(2):
            confmat_data = np.array([[5, 2], [1, 7]])
            df = pd.DataFrame(confmat_data)
            df.to_csv(os.path.join(self.results_dir, f'confmat_seed{i}.csv'), index=False)

    def test_create_title_page(self):
        """Test title page creation"""
        fig = create_title_page()
        self.assertIsInstance(fig, plt.Figure)
        self.assertEqual(len(fig.axes), 1)

    def test_process_seed_data(self):
        """Test processing of seed data"""
        metrics_file = os.path.join(self.results_dir, 'metrics_seed0.json')
        hist_file = os.path.join(self.results_dir, 'history_seed0.csv')
        conf_file = os.path.join(self.results_dir, 'confmat_seed0.csv')

        # Mock PdfPages to capture saved figures
        with patch('matplotlib.backends.backend_pdf.PdfPages') as mock_pdf:
            mock_pdf_instance = MagicMock()
            mock_pdf.return_value.__enter__.return_value = mock_pdf_instance

            process_seed_data(metrics_file, hist_file, conf_file, mock_pdf_instance)

            # Check that savefig was called 3 times (curves, confmat, text)
            self.assertEqual(mock_pdf_instance.savefig.call_count, 3)

    def test_simulate_piezoelectric_sensor(self):
        """Test piezoelectric sensor simulation"""
        # Mock PdfPages
        with patch('matplotlib.backends.backend_pdf.PdfPages') as mock_pdf:
            mock_pdf_instance = MagicMock()
            mock_pdf.return_value.__enter__.return_value = mock_pdf_instance

            fig = simulate_piezoelectric_sensor()

            self.assertIsInstance(fig, plt.Figure)
            self.assertEqual(len(fig.axes), 3)  # Should have 3 subplots

    def test_simulate_piezoelectric_calculations(self):
        """Test the calculations in piezoelectric simulation"""
        # Test parameters
        params = {
            'd33': 2.5e-10,
            'spring_k': 1000,
            'damping_c': 10,
            'capacitance_C': 1e-6,
            'radius': 0.01
        }

        # Test data
        time = np.linspace(0, 1, 100)
        x = 0.01 * np.sin(2 * np.pi * 5 * time)
        dx_dt = np.gradient(x, time)
