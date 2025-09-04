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
# Calculations
        cross_section = np.pi * params['radius'] ** 2
        F_mech = params['spring_k'] * x + params['damping_c'] * dx_dt
        sigma = F_mech / cross_section
        Q = params['d33'] * F_mech
        V = Q / params['capacitance_C']

        # Assertions
        self.assertEqual(len(V), 100)
        self.assertEqual(len(F_mech), 100)
        self.assertEqual(len(sigma), 100)
        self.assertTrue(np.all(np.isfinite(V)))
        self.assertTrue(np.all(np.isfinite(F_mech)))
        self.assertTrue(np.all(np.isfinite(sigma)))

    def test_create_innovation_page(self):
        """Test innovation page creation"""
        fig = create_innovation_page()
        self.assertIsInstance(fig, plt.Figure)
        self.assertEqual(len(fig.axes), 1)

    @patch('matplotlib.backends.backend_pdf.PdfPages')
    @patch('glob.glob')
    def test_generate_report_integration(self, mock_glob, mock_pdf):
        """Test the complete report generation process"""
        # Mock glob to return our test files
        mock_glob.side_effect = [
            sorted([os.path.join(self.results_dir, f'metrics_seed{i}.json') for i in range(2)]),
            sorted([os.path.join(self.results_dir, f'history_seed{i}.csv') for i in range(2)]),
            sorted([os.path.join(self.results_dir, f'confmat_seed{i}.csv') for i in range(2)])
        ]

        # Mock PdfPages
        mock_pdf_instance = MagicMock()
        mock_pdf.return_value.__enter__.return_value = mock_pdf_instance

        # Generate report
        pdf_path = generate_report(self.results_dir)

        # Verify PDF was created
        self.assertTrue(pdf_path.endswith('summary_dual_hemisphere_2031.pdf'))

        # Verify savefig was called multiple times (title + 2 seeds * 3 + sensor + innovations)
        expected_calls = 1 + (2 * 3) + 1 + 1  # title + seeds + sensor + innovations
        self.assertEqual(mock_pdf_instance.savefig.call_count, expected_calls)

    def test_file_reading(self):
        """Test that files can be read correctly"""
        # Test JSON reading
        with open(os.path.join(self.results_dir, 'metrics_seed0.json'), 'r') as f:
            metrics = json.load(f)
        self.assertIn('test_accuracy', metrics)
        self.assertIn('f1_score', metrics)

        # Test CSV reading
        history = pd.read_csv(os.path.join(self.results_dir, 'history_seed0.csv'))
        self.assertIn('epoch', history.columns)
        self.assertIn('accuracy', history.columns)
        self.assertIn('val_accuracy', history.columns)

        confmat = pd.read_csv(os.path.join(self.results_dir, 'confmat_seed0.csv'), index_col=0)
        self.assertEqual(confmat.shape, (2, 2))

if name == '__main__':
    unittest.main()
